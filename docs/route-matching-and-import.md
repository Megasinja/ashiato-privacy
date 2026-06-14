# ASHIATO ルートマッチング & ルートインポート設計メモ

検討日: 2026-06-14

省電力メモ（`battery-optimization.md`）§4-4 のスナップ補正を、道路だけでなく
**鉄道・指定ルート**へ拡張する設計と、外部ルートのインポートについてまとめる。

---

## 1. 移動手段の判別を「車 / 電車」まで細分化する

### 課題

Android の Activity Recognition API は電車も車も `IN_VEHICLE` に
まとめてしまう。**専用の鉄道タイプは存在しない**ため、道路スナップを
そのまま電車区間にかけると線路ではなく並走する道路に吸着してしまう。

### 判別ヒューリスティック（複数を重み付けで合成）

| 手がかり | 電車らしさの判定 |
|---------|----------------|
| **線路への近接** | OSM `railway=rail` ジオメトリから常に 20m 以内を走行 |
| **駅ポリゴンの通過** | 出発・到着が駅構内（`railway=station` の駅領域）と一致 |
| **速度プロファイル** | 駅間で規則的に加減速、駅で完全停止（数十秒〜数分） |
| **直線性・曲率** | 道路より緩いカーブ、急な右左折がない |
| **平均速度** | 在来線 40〜90km/h、新幹線 150km/h 超は鉄道確定（車では出ない） |
| **GPSロスト区間** | 地下区間・トンネルで信号が途切れ、再取得時に大きく前進している |

```kotlin
// 区間（連続する IN_VEHICLE のサブトラジェクトリ）ごとにスコアリング
data class TransitScore(val isRail: Boolean, val confidence: Float, val lineId: String?)

fun classifyVehicleSegment(seg: List<LocationEntity>, rail: RailIndex): TransitScore {
    val nearRailRatio = seg.count { rail.distanceToNearestRail(it) < 20.0 } / seg.size.toFloat()
    val maxSpeed      = seg.maxOf { it.speedMps } * 3.6 // km/h
    val startInStation = rail.stationAt(seg.first())
    val endInStation   = rail.stationAt(seg.last())
    val stopAtStations = detectRegularStops(seg, rail) // 駅位置での停止回数

    var score = 0f
    if (nearRailRatio > 0.8f) score += 0.4f
    if (startInStation != null && endInStation != null) score += 0.25f
    if (stopAtStations >= 1) score += 0.2f
    if (maxSpeed > 130) score += 0.3f      // 在来線特急〜新幹線域

    val lineId = if (score >= 0.5f) rail.dominantLine(seg) else null
    return TransitScore(isRail = score >= 0.5f, confidence = score.coerceAtMost(1f), lineId = lineId)
}
```

ポイント:
- **区間単位（駅to駅 / 乗車to下車）で判定**する。点ごとに判定すると揺れる
- 確信度が低い区間は「補正しない（生データのまま）」にフォールバック。
  誤って線路に吸着させるより、生のGPSのほうがマシ
- GTFS（後述）があれば路線・駅が正確に分かるので判定精度が大きく上がる

---

## 2. 線路へのスナップ（鉄道マップマッチング）

### データソース

| ソース | 内容 | ライセンス | 備考 |
|--------|------|-----------|------|
| **OpenStreetMap** `railway=rail` | 線路ジオメトリ | ODbL | 無料。日本の在来線・新幹線をほぼ網羅 |
| **OSM** `railway=station/halt` | 駅位置・駅構内 | ODbL | 駅ポリゴンで乗降判定 |
| **GTFS / GTFS-JP** | 路線・駅・時刻表 | 事業者により異なる | 国交省「標準的なバス/鉄道情報」。正確な路線形状 |
| **国土数値情報** 鉄道データ | 路線・駅 | 政府標準利用規約 | 日本特化で高品質 |

ODbL（OSM）を使う場合は**帰属表示（Attribution）と、加工データ公開時の
ShareAlike 条項**に注意。アプリ内表示の帰属でほぼ問題ないが、補正済み
ジオメトリを外部配布する場合は要確認。

### アルゴリズム: HMM ベースのマップマッチング

道路でも鉄道でも標準は **隠れマルコフモデル（HMM）+ Viterbi** による
マップマッチング（Newson & Krumm 2009 の手法）。

```
観測 = GPS点列
隠れ状態 = 各GPS点が「どの線路セグメントのどこにいるか」の候補
emission確率 = GPS点と線路候補点の距離（ガウシアン）
transition確率 = 線路ネットワーク上の移動距離とGPS直線距離の整合性
→ Viterbi で最尤の状態系列（=通った線路）を復元
```

実装の選択肢:

| 方法 | 長所 | 短所 |
|------|------|------|
| **OSRM**（鉄道プロファイル） | 実績のあるOSS、`/match` API | 鉄道用プロファイルを自前ビルドする必要 |
| **Valhalla** Meili | マルチモーダル対応のmap-matching | セルフホスト構成が必要 |
| **GraphHopper** map-matching | Java実装、Androidに組込みやすい | 鉄道グラフの構築が必要 |
| **自前 HMM 実装** | 鉄道に最適化・依存なし | 実装コスト大 |

推奨: サーバー側（§battery 4-1 の SyncWorker と同じバックグラウンド処理）で
Valhalla Meili か OSRM を鉄道グラフで動かし、補正済みジオメトリを
別カラムに保存。端末では補正済み座標を表示に使う。

```kotlin
// 区間が鉄道判定なら線路マッチング、車なら道路マッチング、徒歩は生のまま
suspend fun snapSegment(seg: List<LocationEntity>, score: TransitScore): List<LatLng> = when {
    score.isRail && score.confidence > 0.6f -> railMatcher.match(seg, score.lineId)
    score.isVehicle                         -> roadMatcher.match(seg)   // §battery 4-4
    else                                    -> seg.map { it.toLatLng() } // 徒歩・低確信度
}
```

---

## 3. 検索したルート / 指定ルートを「正解の線」として設定する

ユーザーが「この区間はこのルートを通った」と分かっている場合、
GPSをそのルートにスナップさせると最も綺麗になる。

### ユースケース

1. **事後**: 記録を見て「ここは○○線で帰った」とユーザーが路線を指定 → その路線にスナップ
2. **事前**: 出発前にルートを検索・指定しておく → 記録をそのルートにマッチング
3. **インポート**: 外部で作ったルート（GPX/KML）を「基準ルート」として読み込む（§4）

### 実装

基準ルート（polyline）が与えられたら、各GPS点を**そのpolyline上の
最近傍点に射影**するだけでよい（ネットワーク探索不要なので軽い）。

```kotlin
// 基準ルートにGPS点列をスナップ（単純な最近傍射影 + 単調性保証）
fun snapToReferenceRoute(gps: List<LatLng>, route: List<LatLng>): List<LatLng> {
    var lastProj = 0.0 // ルート始点からの累積距離。後戻りを防ぐため単調増加に
    return gps.map { p ->
        val (proj, distAlong) = nearestPointOnPolyline(p, route, minDistAlong = lastProj)
        lastProj = distAlong
        proj
    }
}
```

UI 案:
- 記録の区間を選択 → 「ルートを指定」→ 路線検索 or 保存済みルートから選択
- スナップ前後をプレビュー表示し、ユーザーが採用/取消を選べる
- **元の生データは必ず保持**し、補正は「表示用レイヤー」として上書きしない

### 注意: Directions API のルートをそのまま保存できるか

ユーザーが**アプリ内でルート検索**する場合、バックエンドの規約に注意:
- **Google Directions API**: 経路ジオメトリの**恒久保存は不可**（パフォーマンス
  目的の30日以内の一時キャッシュのみ可、place_idは無期限）。基準ルートとして
  永続保存する用途には使えない
- **OpenRouteService / GraphHopper / Valhalla（OSS, OSM/ODbL）**: 帰属表示と
  ShareAlikeを守れば保存可。検索ルートを基準線に使える。**これを推奨**
- **駅すぱあと API / NAVITIME API**: 日本の公共交通（電車・バス）対応。契約に
  従い保存可。鉄道区間の基準ルート取得に最適
- **Mapbox Directions**: 規約に従えば保存可（要確認）

> ルート検索・インポートの実調査は別途 `routing-api-options.md` を参照。
> Yahoo、駅すぱあと、NAVITIME 等の国内サービスの比較を含む。

---

## 4. 外部ルートのインポート

### 4-1. Google マップからのインポート — 規約上の現実

「Googleマップからルートをインポート」は、経路によって可否がはっきり分かれる。

| 取り込み元 | 可否 | 理由・方法 |
|-----------|------|-----------|
| **Google Takeout（自分のロケーション履歴 / タイムライン）** | ✅ 可 | 自分のデータのエクスポート。KML / JSON で出力でき、取り込み可 |
| **Google マイマップ** | ✅ 可 | KML / KMZ でエクスポート可能。ユーザーが作った地図データ |
| **共有された経路URL（goo.gl/maps 等）の自動解析** | ⚠️ 非推奨 | スクレイピングは規約違反リスク。安定もしない |
| **Directions API の経路を恒久保存** | ❌ 不可 | Google Maps Platform 規約: 経路は30日以内の一時キャッシュのみ可、恒久保存は不可（place_idは無期限） |
| **Yahoo!地図 / 乗換案内のルート** | ⚠️ ほぼ不可 | GPX/KMLの公式エクスポートが無い。API連携で代替する |

**結論**: 「Googleマップからのインポート」は
**①Google Takeout（自分の履歴）と ②マイマップの KML/KMZ** に対応する形にする。
これらは規約上クリーンで、ユーザーが自分のデータを持ち出す正当な経路。
Directions API の経路保存はできないので、ルート検索機能は OSS バックエンド（§3）で代替する。

### 4-2. 対応すべきインポート形式

| 形式 | 用途 | 拡張子 |
|------|------|--------|
| **GPX** | 最も汎用的な軌跡/ルート交換形式。他アプリ（ヤマレコ、Strava等）とも互換 | `.gpx` |
| **KML / KMZ** | Google マイマップ・Earth のエクスポート。KMZ は KML の zip | `.kml` `.kmz` |
| **Google Takeout JSON** | 自分のロケーション履歴 | `.json` |
| **GeoJSON** | 汎用。既に viewer 対応済み | `.geojson` |

### 4-3. パース実装スケッチ

```kotlin
sealed interface ImportedTrack { val points: List<TrackPoint> }
data class TrackPoint(val lat: Double, val lng: Double, val time: Long?, val ele: Double?)

object RouteImporter {
    fun import(uri: Uri, mime: String, bytes: ByteArray): ImportedTrack = when {
        mime.contains("gpx")  || uri.path?.endsWith(".gpx") == true  -> parseGpx(bytes)
        uri.path?.endsWith(".kmz") == true                           -> parseKml(unzipKml(bytes))
        mime.contains("kml")  || uri.path?.endsWith(".kml") == true  -> parseKml(bytes)
        uri.path?.endsWith(".json") == true                          -> parseTakeoutOrGeoJson(bytes)
        else -> throw UnsupportedFormatException(mime)
    }

    // GPX: <trkpt lat lon><time><ele> または <rtept>（ルート点）
    private fun parseGpx(bytes: ByteArray): ImportedTrack { /* XmlPullParser で trkpt/rtept を走査 */ }

    // KML: <coordinates>lon,lat,ele ...</coordinates>（LineString / Track）
    private fun parseKml(bytes: ByteArray): ImportedTrack { /* coordinates をパース。順序は lon,lat,ele */ }

    // KMZ: zip 内の doc.kml を取り出す
    private fun unzipKml(bytes: ByteArray): ByteArray { /* ZipInputStream で *.kml を探す */ }
}
```

注意点:
- **KML の座標順は `経度,緯度,標高`**（GeoJSON と同じ lon,lat 順）。GPX は属性で `lat`/`lon`
- Takeout の `Records.json` は緯度経度が `latE7`（×10^7 の整数）形式。`/1e7` で戻す
- インポートしたルートは**「基準ルート」として §3 のスナップに使う**か、
  **別レイヤーで重ね表示**する（自分の実測軌跡と区別できるよう色を変える）
- 巨大ファイル（数万点）対策にストリーミングパース + §battery 4-2 のバッチ書き込みを流用

### 4-4. プライバシー上の追記事項

外部インポート機能を入れる場合、プライバシーポリシーに以下の趣旨を追記:
- 「利用者が読み込んだ GPX / KML / Takeout データは端末内で処理され、
  利用者本人のアカウントにのみ紐づけて保存される」
- インポート元（Google Takeout 等）のデータ取り扱いは各サービスのポリシーに従う旨

---

## 実装の優先順位（このテーマ内）

| 優先度 | 内容 | 備考 |
|--------|------|------|
| 1 | **GPX / KML(Z) インポート** | 規約クリーン・汎用。まず入口を作る |
| 2 | **基準ルートへの最近傍スナップ（§3）** | ネットワーク探索不要で軽い。インポート or 手動指定と直結 |
| 3 | **車/電車の区間判別（§1）** | 線路スナップの前提。ヒューリスティック→GTFSで精度向上 |
| 4 | **鉄道マップマッチング（§2）** | サーバー側 Valhalla/OSRM。一番重い。最後 |
| – | Google Takeout JSON 取り込み | 1 と同時か直後。自分の履歴移行ニーズが高ければ前倒し |

> ルート検索を自前で持つ場合は Directions API ではなく
> **OSS ルーティング（OSRM / Valhalla / GraphHopper）** を使う。
> Google の経路は規約上アプリ内に保存・表示できないため。
