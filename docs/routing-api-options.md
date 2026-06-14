# ASHIATO ルート検索・ルートインポート 調査結果

調査日: 2026-06-14  
目的: 「検索したルートを軌跡の基準線に使う」「外部のルートを取り込む」機能の
実現方法を、規約・保存可否・日本対応の観点で実調査した結果。

> 本ドキュメントは Web 調査に基づく。各APIの料金・規約は変動するため、
> 実装前に各公式ページで最新条件を必ず確認すること。出典は末尾に記載。

---

## 0. 結論（先に要点）

| やりたいこと | 推奨手段 | 理由 |
|------------|---------|------|
| 車・徒歩・自転車のルート検索＋**保存**して基準線に使う | **OpenRouteService**（OSS/ODbL） | 結果を保存できる（Googleは不可）。自前ホスト可 |
| 電車・バスのルート（鉄道区間スナップ用） | **駅すぱあと API** or **NAVITIME API** | 日本の公共交通に対応。Yahoo乗換案内の中身も駅すぱあと |
| 既存ルートのインポート | **GPX / KML(Z)** 取り込み | マイマップ・Takeout・各種アプリと互換。実装済み |
| Googleマップ由来のルート取り込み | マイマップ→KML / Takeout→KML/JSON のみ | Directions APIの結果は恒久保存不可（規約） |

**重要な訂正**: 以前「Google の経路は保存できない」と説明したが、正確には
**「パフォーマンス目的の30日以内の一時キャッシュは可、恒久保存は不可」**
（place_id は無期限保存可）。いずれにせよ「基準ルートとして永続保存」には使えない。

---

## 1. ルート検索 API の比較（車・徒歩・自転車）

| プロバイダ | 対応モード | ジオメトリ取得 | 結果の保存 | 料金 | 日本 |
|-----------|----------|--------------|-----------|------|------|
| **Google Directions** | 車/徒歩/自転車/公共交通 | polyline | ❌ 恒久不可（30日キャッシュのみ） | 従量課金 | ◎ |
| **Yahoo YOLP 経路探索/経路地図** | 車/徒歩 | 経路地図(画像) / JS地図上のジオメトリ | 要Yahoo規約確認 | 無料枠5万/日 | ◎（国内のみ） |
| **OpenRouteService** | 車/徒歩/自転車 | GeoJSON（座標列） | ✅ ODbL（帰属＋ShareAlike） | 無料 / 自前ホスト | ○（OSM由来） |
| **GraphHopper** | 車/徒歩/自転車 | 座標列 | ✅ 自前ホスト(OSM) | 自前ホスト無料 / SaaS有料 | ○ |
| **Valhalla** | 車/徒歩/自転車 | 座標列＋**map-matching対応** | ✅ 自前ホスト(OSM) | 自前ホスト | ○ |
| **Mapbox Directions** | 車/徒歩/自転車 | polyline | 要Mapbox規約確認 | 従量課金 | ○ |
| **NAVITIME API** | 車/徒歩/自転車/**公共交通** | あり | 契約による | 約$200/月/機能〜（RapidAPI） | ◎ |

### 1-1. Google Directions API — 保存制限が致命的

- 経路ジオメトリは**恒久保存不可**。キャッシュは「ネットワーク遅延対策で30日以内・
  非再配布・暗号化保存」などの条件付きのみ。place_id だけ無期限保存可。
- ASHIATO の「基準ルートを保存して後で軌跡をスナップ」用途には**使えない**。
- ルート検索の品質は高いので、「その場で表示するだけ（保存しない）」なら選択肢にはなる。

### 1-2. Yahoo YOLP — 国内特化だが経路は画像/JS寄り

- **経路地図API (routemap)**: 出発地→目的地の経路を描いた**地図画像**を返す。
  `search:off` で自前の経路データを渡して描画だけさせることも可能。
- **経路探索**: JavaScriptマップAPI上で経路探索を行うサンプルがある（地図表示前提）。
- いずれも「polyline ジオメトリを取り出して保存し、ネイティブアプリでスナップに使う」
  という用途には素直に向かない（画像・JS地図表示が主目的）。
- 無料枠は大きい（5万アクセス/日）が、**公共交通（乗換）はYOLPに含まれない**。
- 利用には Yahoo! JAPAN ID と clientID 取得が必要。商用利用・保存は要規約確認。

### 1-3. OpenRouteService（ORS）— 保存できる本命

- OpenStreetMap ベース。Heidelberg大学 GIScience が開発。**GPL-3.0 / OSS**。
- `/directions` が **GeoJSON で座標列**を返すので、そのまま基準ルートに使える。
- **ODbL ライセンス**: 商用利用OK。ただし
  - **帰属表示（Attribution）必須**（「© OpenStreetMap contributors」等）
  - **ShareAlike**: 加工したDBを**公開**する場合は同ライセンスで提供する義務。
    → 利用者本人だけが見る（Firestoreで本人限定）私的保存は「公開」に当たらないと
      解釈できるが、**法的判断は要確認**。
- ホスト型の無料APIは利用制限あり（1日あたりのリクエスト上限）。
  本番は **Docker で自前ホスト**して制限とプライバシーを自分でコントロールするのが堅い。

### 1-4. GraphHopper / Valhalla — 自前ホスト＋map-matching

- どちらもOSSでOSMベース。自前ホストすれば結果保存に制約が少ない（OSMのODbL帰属は必要）。
- **Valhalla の Meili** はmap-matching（GPS→道路/任意ネットワークへのスナップ）も担えるので、
  §route-matching の道路・鉄道スナップと**同じ基盤で実装できる**のが利点。
- 「ルート検索」と「軌跡スナップ」を1つのサーバーで賄いたいなら Valhalla が有力。

---

## 2. 公共交通（電車・バス）のルート — 日本の事情

### 2-1. Yahoo!乗換案内の API は公開されていない

- **Yahoo!乗換案内そのものの公開APIは無い**。アプリ・Web向けの提供のみ。
- そして **Yahoo!乗換案内の経路エンジンは「駅すぱあと」（ヴァル研究所）が提供**している。
  → 公共交通APIが欲しいなら、Yahooの裏側にいる**駅すぱあとを直接使う**のが筋。

### 2-2. 駅すぱあと API（旧:駅すぱあとWebサービス）

- 2024年2月に「駅すぱあとWebサービス」→「**駅すぱあと API**」へ名称変更。
- 経路探索・乗換案内・運賃/定期/交通費計算・路線/駅情報を提供。導入12万社以上。
- **完全無料プラン**（個人・商用可、機能制限あり）／リクエスト買い切り型／従量制。
- 事前登録不要の **Playground** で試せる。
- 鉄道区間のスナップ（§route-matching §5-4）に必要な**駅・路線データの正確な供給源**。

### 2-3. NAVITIME API

- 車/徒歩/自転車/バイク/**公共交通（トータルナビ）**をまとめて持つ国内最強の多モーダル。
- 大型車ルート・休憩地点提案・VICS渋滞・バス時刻表などオプション豊富。
- 料金は RapidAPI 経由で **PROプラン月額$200/機能**（地図＋ルートなら$400）。
  直接契約は **90日無料トライアル**、APIマーケットは500アクセスまで無料枠。

### 2-4. 公共交通の使いどころ

ASHIATO で電車区間を「線路に沿わせる」には、線路ジオメトリと駅位置が要る（§route-matching §5-5）。
選択肢は2系統:
- **無料・OSS路線**: OSM `railway=rail` ＋ 国土数値情報（鉄道）＋ GTFS-JP。自前で持つ。
- **商用API路線**: 駅すぱあと/NAVITIME で経路（駅to駅）を取得し、その区間を基準ルートにスナップ。

ユーザーが「○○線で帰った」と路線名を指定できるなら、駅すぱあとで該当区間の
経路ジオメトリを取得 → §route-matching §5-3 の最近傍スナップに渡すのが最も綺麗。

---

## 3. ルートのインポート（外部ルートの取り込み）

### 3-1. 取り込み元ごとの可否

| 取り込み元 | 形式 | 可否 | 方法 |
|-----------|------|------|------|
| **Googleマイマップ** | KML / KMZ | ✅ | マップの︙→「KML/KMZにエクスポート」。KMLにチェックで出力 |
| **Google タイムライン / Takeout** | KML / JSON | ✅ | Takeoutでロケーション履歴をエクスポート |
| **登山・自転車アプリ**（ヤマレコ/Strava/RouteShare等） | GPX | ✅ | 各アプリのGPXエクスポート |
| **GISソフト**（QGIS/ArcGIS） | KML / GeoJSON | ✅ | 標準エクスポート |
| **Yahoo!地図 / Yahoo!乗換案内** | – | ⚠️ ほぼ不可 | GPX/KMLの公式エクスポートが無い。URL共有のみ |
| **Google Directions API の経路** | – | ❌ | 恒久保存が規約違反 |

### 3-2. Yahoo からのインポートは弱い

- Yahoo!地図・乗換案内には**GPX/KMLでルートを書き出す公式機能が無い**。
- したがって「Yahooで検索したルートをファイルで取り込む」は現実的でない。
- Yahooを使うなら**インポートではなくAPI連携**（§1-2 / §2）で、アプリ内検索として
  組み込む方向になる。ただしYOLPは画像/JS寄りで公共交通も無いため、
  日本の交通用途では**駅すぱあと/NAVITIME**のほうが実用的。

### 3-3. KML パースの注意（再掲）

- KMLの座標順は **`経度,緯度,標高`**（GPXの `lat`/`lon` 属性とは逆）。
- マイマップのエクスポートは KMZ（zip）になることがある → 解凍して `doc.kml` を読む。
- 複数 KML の同時取り込みでエラーになる事例あり → 1ファイルずつ処理。
- `gx:Track`（時刻付き）と通常の `<coordinates>` の両方に対応する。

### 3-4. 実装状況（Webプロトタイプ trajectory.html）

| 形式 | 取り込み |
|------|---------|
| JSON配列 / GeoJSON(Point/LineString) | ✅ |
| GPX (`trkpt`/`rtept`) | ✅ |
| KML (`coordinates` / `gx:Track`) | ✅ |
| Google Takeout JSON (`latitudeE7`) | ✅ |
| KMZ（zip解凍） | △ 未対応（要zip展開、Android側で実装） |

---

## 4. ライセンス・規約の早見表

| サービス | 結果の保存 | 帰属表示 | 商用 | 備考 |
|---------|-----------|---------|------|------|
| Google Directions | ❌恒久不可（30日キャッシュ可、place_idは無期限） | 必要 | 可 | 基準ルートの永続保存に不適 |
| Yahoo YOLP | 要確認 | 必要 | 要確認 | 画像/JS地図前提、公共交通なし |
| OpenRouteService (ODbL) | ✅（公開時ShareAlike） | 必要 | 可 | OSM由来。私的保存は概ね可 |
| GraphHopper/Valhalla (OSM) | ✅自前ホスト | 必要 | 可 | map-matchingも可 |
| 駅すぱあと API | 契約による | 規定あり | 可 | 無料プランあり。公共交通の本命 |
| NAVITIME API | 契約による | 規定あり | 可 | 多モーダル最強・高価 |

---

## 5. ASHIATO への推奨アーキテクチャ

```
┌─ ルート検索（車/徒歩/自転車） ─ OpenRouteService（自前ホスト or 無料枠）
│     → 結果を「基準ルート」としてFirestoreに保存（本人限定・ODbL帰属表示）
│
├─ ルート検索（電車/バス）       ─ 駅すぱあと API（無料プランで開始）
│     → 駅to駅の経路を取得し、IN_VEHICLE区間の基準ルートに
│
├─ 軌跡スナップ                  ─ Valhalla Meili（道路）/ 上記基準ルートへ最近傍射影
│     → §route-matching-and-import.md の §3, §5 と統合
│
└─ インポート                    ─ GPX / KML / KMZ / Takeout（ファイル取り込み）
      → Yahooはエクスポート機能が無いため対象外。API連携で代替
```

### 段階的導入

| Phase | 内容 |
|-------|------|
| 1 | GPX/KML/KMZ インポート（依存なし・規約クリーン） |
| 2 | OpenRouteService 連携（車/徒歩の基準ルート検索＋保存） |
| 3 | 駅すぱあと API 連携（電車ルート＋駅/路線データ） |
| 4 | Valhalla で道路・鉄道スナップを統合 |

---

## 6. 出典

- [YOLP(地図) 経路地図API リファレンス](https://developer.yahoo.co.jp/webapi/map/openlocalplatform/v1/routemap.html)
- [YOLP JavaScriptマップAPI 経路探索サンプル](https://yahoojapan.github.io/yolp-jsapi-samples/service-routesearch.html)
- [YOLP 一部API・SDK提供終了のお知らせ](https://map.yahoo.co.jp/blog/archives/20200116_yolp_close.html)
- [NAVITIME API 仕様書（API一覧と料金）](https://api-sdk.navitime.co.jp/api/specs/description/about_navitime_api.html)
- [NAVITIME API トータルナビ（公共交通ルート検索）](https://api-sdk.navitime.co.jp/api/specs/api_guide/route_transit.html)
- [駅すぱあと API 公式](https://api-info.ekispert.com/)
- [駅すぱあとWebサービス→駅すぱあとAPI 名称変更（ヴァル研究所）](https://www.val.co.jp/topics/20240222)
- [Google Maps Platform サービス別規約（キャッシュ条件）](https://cloud.google.com/maps-platform/terms/maps-service-terms)
- [Directions API ポリシー](https://developers.google.com/maps/documentation/directions/policies)
- [OpenRouteService 公式](https://openrouteservice.org/)
- [OpenRouteService 利用規約](https://openrouteservice.org/terms-of-service/)
- [ODbL v1.0（Open Data Commons）](https://opendatacommons.org/licenses/odbl/1-0/)
- [Googleマイマップ KMLエクスポート手順](http://gmap.pw/mymap/export_xml/)
- [GPX/KMLからルートをインポート（RouteShare）](https://route-share.net/blogs/articles/5)
