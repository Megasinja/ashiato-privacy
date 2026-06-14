# ASHIATO Android 開発メモ — 設計・考慮事項まとめ

作成日: 2026-06-14  
対象: ASHIATO Android アプリ（GPS移動履歴記録・閲覧アプリ）

---

## 目次

1. [アプリ概要](#1-アプリ概要)
2. [GPS軌跡の再生表示](#2-gps軌跡の再生表示)
3. [バックグラウンドGPS省電力化](#3-バックグラウンドgps省電力化)
4. [軌跡品質の向上（スナップ・後処理）](#4-軌跡品質の向上)
5. [外部ルートのインポートと鉄道対応](#5-外部ルートのインポートと鉄道対応)
6. [軌跡の手動補正](#6-軌跡の手動補正)
7. [写真とGPS軌跡の突合](#7-写真とgps軌跡の突合)
8. [実装優先順位（全体）](#8-実装優先順位全体)
9. [未確定事項・今後の調査項目](#9-未確定事項今後の調査項目)

---

## 1. アプリ概要

| 項目 | 内容 |
|------|------|
| アプリ名 | ASHIATO |
| 機能概要 | バックグラウンドでGPSを記録し、地図上で移動履歴を振り返る |
| ストレージ | 端末内 Room (SQLite) + Google Firebase / Cloud Firestore |
| 認証 | Google Sign-In |
| 対象OS | Android（iOS も対象かは確認中） |
| 言語 | Kotlin（推定） |

---

## 2. GPS軌跡の再生表示

### 2-1. 2つの再生モード

軌跡を地図上で再生するとき、2種類のモードを提供する。

#### モード A：時間速度一定（実時間再生）

- 実際の経過時間を一定倍率（例: 60×）で圧縮して再生
- GPS点の訪問順序は記録した時刻通り
- **止まっていた時間・滞在時間もそのまま反映される**
- ユーザーが「いつ・どこにいたか」を時系列で確認するのに適している
- 倍率範囲: 1× 〜 3600×（対数スケールのスライダーで操作）

#### モード B：移動速度一定（距離再生）

- マーカーがルート上を一定の空間速度（例: 50 m/playback秒）で進む
- 速歩き・ジョギング・立ち止まりを問わず、ルートの全区間を均等な画面時間で表示
- **止まっていた時間が長くても、その場所でつまらない再生にならない**
- どのルートを通ったかを確認するのに適している
- 速度範囲: 1 〜 2500 m/s（対数スケール）

#### 実装の核心：進捗度 0〜1 での統一的な補間

```kotlin
// progress: 0.0 (開始) 〜 1.0 (終了)
// モードAなら progress = playbackElapsed * speedMultiplier / totalRealTime
// モードBなら progress = playbackElapsed * metersPerSec / totalDistanceMeters

fun stateAt(progress: Double, mode: PlayMode): TrackState {
    return when (mode) {
        PlayMode.TIME -> {
            val targetMs = progress * totalTime
            val idx = cumTime.binarySearch(targetMs)    // 二分探索
            interpolate(idx, targetMs, cumTime)
        }
        PlayMode.DISTANCE -> {
            val targetM = progress * totalDist
            val idx = cumDist.binarySearch(targetM)
            interpolate(idx, targetM, cumDist)
        }
    }
}
```

#### 表示要素

- 未通過ルート: 薄いグレーの polyline
- 通過済みルート: テールカラーの polyline（リアルタイム更新）
- 現在地マーカー: アニメーションするパルスドット
- スタート: 緑のドット / ゴール: 赤のドット
- 情報パネル: 現在時刻・経過時間・移動距離・速度（リアルタイム）

### 2-2. 累積データの前処理

再生前に1回だけ前処理しておく（毎フレーム計算しない）:

```kotlin
data class TrackMeta(
    val cumDist: FloatArray,   // 各点までの累積距離(m)
    val cumTime: LongArray,    // 各点までの経過時間(ms)
    val totalDist: Float,
    val totalTime: Long
)

fun preprocess(points: List<TrackPoint>): TrackMeta {
    // O(n) で累積計算。これを二分探索に使う
}
```

---

## 3. バックグラウンドGPS省電力化

### 3-1. 問題の構造

バックグラウンドGPSの電力消費は主に3層から来る:

```
GPSチップ（測位）      ← 最大消費。間隔・精度・静止中停止で大幅削減
  ↓
CPU/メモリ（処理）     ← PendingIntent + バッチで削減
  ↓
モバイルラジオ（通信）  ← Firestore同期をWiFi/充電時にまとめることで削減
```

### 3-2. 静止検知でGPSを止める（効果最大）

**問題**: Activity Recognition API の動き出し検知には30秒〜数分の遅延がある。

**原因**:
1. センサーを周期的にサンプリングするため（常時監視でない）
2. 誤検知防止のデバウンス（複数サンプルで一貫した結果が出るまで待つ）
3. Doze モードでの配信遅延

**解決策: Significant Motion + Activity Transition の役割分担**

```
静止中: GPS停止 + Significant Motion センサー待機
         ↓（数秒で発火）
  動き出しを検知 → GPS即開始 + lastLocation を先頭点として記録（出発点の補完）
         ↓（〜30秒後）
  Activity Transition API が移動手段を分類
         ↓
  移動手段に応じたLocationRequestに切り替え
         ↓
  Activity Transition が STILL を検知 → GPS停止
  （STILL は3分継続で判定。信号待ち・コンビニ立ち寄りで切れないよう余裕を持たせる）
```

```kotlin
// Significant Motion: ハードウェアセンサーハブで動く、Doze中でも発火
val sensorManager = getSystemService(SensorManager::class.java)
val sigMotion = sensorManager.getDefaultSensor(Sensor.TYPE_SIGNIFICANT_MOTION)
sensorManager.requestTriggerSensor(object : TriggerEventListener() {
    override fun onTrigger(event: TriggerEvent) {
        startLocationUpdates()
        recordLastKnownLocation() // 出発点の取りこぼし防止
        // ワンショットなので静止時に再登録
    }
}, sigMotion)
```

**権限**: `ACTIVITY_RECOGNITION`（Android 10+）が追加で必要。プライバシーポリシーに追記。

### 3-3. 前面/背面で取得方法を切り替える

**方針**: 測位周期は変えず、配信方法だけ切り替える。

| 状態 | 配信方法 | 理由 |
|------|---------|------|
| バックグラウンド | `PendingIntent` + `setMaxUpdateDelayMillis(60_000)` | プロセスが起きっぱなしにならない |
| フォアグラウンド | `LocationCallback` + `setMaxUpdateDelayMillis(0)` | 地図にリアルタイム表示 |

```kotlin
ProcessLifecycleOwner.get().lifecycle.addObserver(LifecycleEventObserver { _, event ->
    when (event) {
        Lifecycle.Event.ON_START -> {
            fusedClient.flushLocations() // 溜まったバッチを即吐き出し → 地図が最新になる
            fusedClient.removeLocationUpdates(locationPendingIntent)
            fusedClient.requestLocationUpdates(fgRequest, callback, Looper.getMainLooper())
        }
        Lifecycle.Event.ON_STOP -> {
            fusedClient.removeLocationUpdates(callback)
            fusedClient.requestLocationUpdates(bgRequest, locationPendingIntent)
        }
        else -> {}
    }
})
```

**注意**: `removeLocationUpdates` → `requestLocationUpdates` の順を守ること（二重登録防止）。

### 3-4. 移動手段別の動的LocationRequest

Activity Transition API の分類結果を流用して間隔を自動調整。
「1点あたりの移動距離をだいたい揃える」が設計の指針。

| 移動手段 | 取得間隔 | 距離フィルタ | バッチ上限(背面) | 精度 |
|---------|---------|------------|----------------|------|
| STILL | **GPS停止** | – | – | – |
| WALKING | 20秒 | 15 m | 120秒 | HIGH |
| RUNNING | 8秒 | 20 m | 60秒 | HIGH |
| ON_BICYCLE | 5秒 | 25 m | 60秒 | HIGH |
| IN_VEHICLE | 4秒 | 50 m | 30秒 | BALANCED可 |
| UNKNOWN | 10秒 | 10 m | 60秒 | HIGH |

```kotlin
fun onTransition(activityType: Int) {
    val req = when (activityType) {
        DetectedActivity.STILL      -> null  // GPS停止
        DetectedActivity.WALKING    -> buildRequest(20_000, 15f, 120_000)
        DetectedActivity.RUNNING    -> buildRequest(8_000, 20f, 60_000)
        DetectedActivity.ON_BICYCLE -> buildRequest(5_000, 25f, 60_000)
        DetectedActivity.IN_VEHICLE -> buildRequest(4_000, 50f, 30_000)
        else                        -> buildRequest(10_000, 10f, 60_000)
    }
    swapLocationRequest(req)
}

fun buildRequest(intervalMs: Long, minDistM: Float, batchMs: Long) =
    LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, intervalMs)
        .setMinUpdateDistanceMeters(minDistM)
        .setMaxUpdateDelayMillis(if (isForeground) 0L else batchMs)
        .build()
```

**注意点**:
- 遷移頻発防止のため、同じ手段が続く間は再リクエストしない
- 遷移から数十秒はクールダウン（再切り替えしない）
- Activity Recognitionが「電車も車も `IN_VEHICLE`」にまとめるため、電車判定は別途必要（§5参照）

### 3-5. Firestore同期を充電中・WiFi時に集約

GPSの次に電池を食うのがモバイル通信のラジオ起動。1点書き込むたびに
セルラーラジオが10〜20秒間起動し続ける（Radio Tail）。

```kotlin
val syncWork = PeriodicWorkRequestBuilder<FirestoreSyncWorker>(6, TimeUnit.HOURS)
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.UNMETERED) // WiFi
        .setRequiresCharging(true)                     // 充電中
        .build())
    .build()
WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork("firestore_sync", ExistingPeriodicWorkPolicy.KEEP, syncWork)
```

- 前面時のみ即時同期（ON_START で `syncToFirestoreNow()`）
- Firestoreへの書き込みは500件単位のバッチコミット
- `setRequiresStorageNotLow()` でディスク不足時のエラーを防ぐ

**考慮点**: WiFi+充電の両条件は厳しすぎて数日同期されない場合がある。
「WiFi **または** 充電中 かつ 未同期が N 点以上」などに緩める選択肢もある。

### 3-6. ローカルDB書き込みのバッチ化

```kotlin
class LocationBuffer(
    private val dao: LocationDao,
    private val maxSize: Int = 30,        // 30点溜まったらフラッシュ
    private val maxAgeMs: Long = 60_000   // 最大60秒に1回は必ずフラッシュ
) {
    fun add(point: LocationEntity) {
        buffer.add(point)
        if (buffer.size >= maxSize || System.currentTimeMillis() - lastFlushAt >= maxAgeMs)
            flush()
    }
    fun flush() { /* Room の insertAll でトランザクション一括INSERT */ }
}
```

フォアグラウンドサービスの `onDestroy` で必ず `flush()` を呼ぶこと。

### 3-7. バッテリーセーバー連動（オプション）

```kotlin
// 残量15%以下で省電力モードへ自動切り替え
class BatteryReceiver : BroadcastReceiver() {
    override fun onReceive(ctx: Context, intent: Intent) {
        val pct = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) * 100 /
                  intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
        val mode = when { pct <= 15 -> POWER_SAVE; pct <= 30 -> BALANCED; else -> NORMAL }
        LocationSettingsManager.applyMode(mode)
    }
}
// システムの Battery Saver ON/OFF にも連動
registerReceiver(receiver, IntentFilter(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED))
```

UX注意: ユーザーが知らないうちに精度が落ちるのは避ける。
通知で伝えるか、アプリ設定で「省電力/標準/高精度」の3段階をユーザーが選べるようにする。

### 3-8. 省電力化の優先順位

| 優先度 | 内容 | 効果 | コスト |
|--------|------|------|--------|
| 1 | 前面/背面切り替え + PendingIntent化 | 基盤 | 中 |
| 2 | Significant Motion + Transition API による静止GPS停止 | ★★★ | 中 |
| 3 | 移動手段別の動的間隔 | ★★★ | 小（§2流用） |
| 4 | ローカルDB バッチ書き込み | ★★ | 小 |
| 5 | Firestore同期のWorkManager化 | ★★ | 中 |
| 6 | バッテリーセーバー連動 | ★ | 小 |

---

## 4. 軌跡品質の向上

### 4-1. 外れ値除去

GPS誤差で「一瞬100m飛んだ」ような点を除去する。

```kotlin
fun filterOutliers(points: List<LocationEntity>): List<LocationEntity> {
    val result = mutableListOf<LocationEntity>()
    for (i in points.indices) {
        if (i == 0) { result.add(points[i]); continue }
        val prev = result.last()
        val d  = haversineMeters(prev, points[i])
        val dt = max(1.0, (points[i].timestamp - prev.timestamp) / 1000.0)
        // implied speed が200km/h (55m/s) 超 = GPS誤差とみなして除外
        if (d / dt < 55.0) result.add(points[i])
    }
    return result
}
```

### 4-2. Douglas-Peucker 簡略化

ズームアウト時に点を間引いて描画を軽くする。ズームインでは元の点列を使う。

```kotlin
fun douglasPeucker(points: List<LatLng>, epsilon: Double): List<LatLng> {
    // 線分 first-last から最も遠い点を探し、epsilon超なら再帰分割
    // O(n log n)。ズームレベルに応じてepsilonを変える（例: zoom10→20m, zoom15→2m）
}
```

### 4-3. 道路スナップ（車・自転車区間）

車・自転車判定の区間に対して、GPS点列を道路ネットワークに吸着させる。

**OSS オプション（端末ではなくサーバー側で処理）**:
- OSRM `/match` エンドポイント
- Valhalla Meili（マルチモーダル対応）
- GraphHopper map-matching

**実装方針**:
- 補正済み座標を別カラム（`snapped_lat`, `snapped_lng`）に保存
- 表示時に「生データ」と「補正済み」を切り替えられるようにする
- SyncWorker（§3-5）のタイミングで非同期処理

### 4-4. 後処理補正の注意点

- **徒歩区間はスナップしない**（歩道外・路地・公園内を歩く場合がある）
- **補正は常に生データを上書きしない**（元データを必ず保持）
- 電車区間の道路スナップは誤り（§5の鉄道判別が必要）

---

## 5. 外部ルートのインポートと鉄道対応

### 5-1. 対応すべきインポート形式

| 形式 | 用途 | 注意点 |
|------|------|--------|
| **GPX** | 汎用軌跡交換。ヤマレコ・Strava等 | `trkpt` (軌跡) と `rtept` (ルート) が混在 |
| **KML / KMZ** | Googleマイマップ、Google Earth | **座標順が `経度,緯度,標高`**（lon,lat 順）GPXと逆 |
| **GeoJSON** | Web汎用 | `Point` と `LineString` 両方に対応 |
| **Google Takeout JSON** | 自分のロケーション履歴 | `latitudeE7` / `longitudeE7`（×10^-7 で度に変換） |

**Google Directions API の規約上の制限**:
経路検索結果をGoogleマップ以外の地図に保存・表示することは規約違反。
アプリ内のルート検索には OSS ルーティング（OSRM / Valhalla / GraphHopper）を使う。
インポートできるのは「自分のデータ（Takeout）」と「自分で作ったマイマップKML」のみ。

### 5-2. KMLパース時の注意点

```kotlin
// KML の座標は "経度,緯度,標高" の順（GPXの lat/lon 属性と逆）
// gx:Track 要素（時刻付き）と通常の coordinates 両方に対応する必要あり

// gx:Track の例:
// <when>2024-06-01T09:00:00Z</when>
// <gx:coord>139.69 35.68 0</gx:coord>  ← lon lat alt の順
```

### 5-3. インポートしたルートへのGPSスナップ

「この区間はこのルートを通った」と分かっている場合、
インポートした polyline に GPS点を射影（最近傍点）するだけで補正できる。
ネットワーク探索が不要なので軽量。

```kotlin
fun snapToReferenceRoute(gps: List<LatLng>, route: List<LatLng>): List<LatLng> {
    var lastProjDist = 0.0 // 後戻りを防ぐ（単調増加）
    return gps.map { p ->
        val (proj, distAlong) = nearestPointOnPolyline(p, route, minDistAlong = lastProjDist)
        lastProjDist = distAlong
        proj
    }
}
```

### 5-4. 電車・鉄道区間の判別

Activity Recognition API は電車を `IN_VEHICLE` に含めてしまう。
区間単位でヒューリスティックスコアリングして判別する。

| 判別手がかり | 電車らしさ |
|------------|-----------|
| OSM `railway=rail` から常に20m以内 | 高 |
| 出発・到着が駅構内（`railway=station` ポリゴン） | 高 |
| 速度 130km/h 超（在来線特急・新幹線） | 確定 |
| 駅での規則的な停止 | 中 |
| GPSロスト後に大きく前進（トンネル通過） | 中 |

```kotlin
fun classifyVehicleSegment(seg: List<LocationEntity>): TransitScore {
    var score = 0f
    val nearRailRatio = seg.count { railIndex.distToNearestRail(it) < 20.0 } / seg.size.toFloat()
    val maxSpeed = seg.maxOf { it.speedMps } * 3.6 // km/h
    if (nearRailRatio > 0.8f) score += 0.4f
    if (startInStation && endInStation) score += 0.25f
    if (stopAtStations >= 1) score += 0.2f
    if (maxSpeed > 130) score += 0.3f
    return TransitScore(isRail = score >= 0.5f, confidence = score.coerceAtMost(1f))
}
```

**確信度が低い区間は補正しない**（誤って線路に吸着させるより生データのほうがマシ）。

### 5-5. 地図データソース（鉄道マッチング用）

| ソース | 内容 | ライセンス |
|--------|------|-----------|
| OpenStreetMap `railway=rail` | 線路ジオメトリ | ODbL（帰属表示必須） |
| OSM `railway=station/halt` | 駅位置・駅構内ポリゴン | ODbL |
| 国土数値情報 鉄道データ | 路線・駅（日本特化、高品質） | 政府標準利用規約 |
| GTFS-JP | 路線・駅・時刻表 | 事業者により異なる |

### 5-6. 実装優先順位（このテーマ）

| 優先度 | 内容 |
|--------|------|
| 1 | GPX / KML(Z) インポート |
| 2 | 指定ルートへの最近傍スナップ |
| 3 | 車/電車の区間判別（ヒューリスティック） |
| 4 | 鉄道マップマッチング（サーバー側 Valhalla/OSRM） |

---

## 6. 軌跡の手動補正

### 6-1. 機能の概要

ユーザーが「ここはGPSがずれていた」と分かる場合に手動で修正する。

**2つのアクション**:
1. **自動補正（区間指定）**: 始点・終点を選んで「この区間をきれいにして」と依頼
2. **点の移動**: 特定の1点を正しい位置にドラッグで移動

### 6-2. 自動補正のアルゴリズム

選択区間 `[a, b]` に対して2パス処理:

```kotlin
// Pass 1: 外れ値除去（implied speed > 200 km/h の点を除く）
val keep = mutableListOf(a)
for (i in a+1..b) {
    val prev = points[keep.last()]
    val d  = haversineMeters(prev, points[i])
    val dt = max(1.0, (points[i].timestamp - prev.timestamp) / 1000.0)
    if (d / dt < 55.0) keep.add(i) // 200km/h未満 → 保持
}
if (keep.last() != b) keep.add(b)

// Pass 2: Gaussian smooth（σ=2.5、端点a・bは固定して隣接区間と繋ぎ目が自然になるよう）
val R = ceil(2.5 * sigma).toInt()
for (ki in 1 until keep.size - 1) {
    var wLat = 0.0; var wLng = 0.0; var wSum = 0.0
    for (kj in max(0, ki-R)..min(keep.size-1, ki+R)) {
        val w = exp(-((ki-kj).toDouble().pow(2)) / (2 * sigma.pow(2)))
        wLat += w * points[keep[kj]].lat
        wLng += w * points[keep[kj]].lng
        wSum += w
    }
    points[keep[ki]] = points[keep[ki]].copy(lat = wLat/wSum, lng = wLng/wSum)
}
```

**設計上の重要ポイント**:
- **元データは必ず保持する**（補正は別カラムまたは別テーブルに保存）
- **無制限のUndo**（操作スタック）
- 補正前後のプレビュー表示を提供する

### 6-3. 点の移動

```kotlin
// 1点を選択 → 地図上の新しい座標をタップ → 移動
fun movePoint(index: Int, newLat: Double, newLng: Double) {
    undoStack.push(points.toList()) // Undo用にスナップショット
    points[index] = points[index].copy(lat = newLat, lng = newLng)
    recomputeMeta() // 累積距離・時間を再計算
}
```

### 6-4. UI/UX

```
編集モードボタン（ペンアイコン）をタップ
  → 再生UI非表示 → 編集ツールバー表示
  → ルート上にドット（間引き表示、最大2000点）

[始点をタップ] → オレンジ表示
[終点をタップ] → 赤表示、区間がオレンジにハイライト
[自動補正] → 2パス処理 → ルート更新 → Undoスタックに積む
[点を移動] → 地図クリックで新座標設定
[元に戻す] → 前の状態に戻る
[書き出す] → 補正済みデータをエクスポート
```

**パフォーマンス注意**: 数千点を全てドラッグ可能マーカーにするのはメモリ・描画両方で重い。
表示点数は最大2000に間引きし、ズームインで精細表示するのが現実的。

---

## 7. 写真とGPS軌跡の突合

### 7-1. 紐付けの優先度

```
優先 1: EXIF GPS座標 (GPSLatitude / GPSLongitude) が存在する
   → そのまま地図座標として使用
優先 2: EXIF DateTimeOriginal を軌跡タイムスタンプで補間
   → GPS軌跡上の対応位置を線形補間で算出
```

### 7-2. 最重要の問題：タイムゾーン

| データ | 形式 | 問題 |
|--------|------|------|
| EXIF `DateTimeOriginal` | `2024:06:01 09:15:30`（ローカル時刻、TZ情報なし） | |
| GPS タイムスタンプ | UTC または UNIX ms | |
| → **差分** | JST なら9時間ずれる | **そのまま照合すると9時間ずれた場所になる** |

**対処**:
1. ユーザーが写真のタイムゾーンを指定（デフォルト JST=UTC+9）
2. `EXIF OffsetTimeOriginal` タグがあればそれを使う（対応カメラは少ない）
3. 軌跡の先頭時刻と写真の時刻を比較してオフセットを自動推定する方法もある

**クロックドリフト対策**:
カメラの時計と実時刻が数十秒ずれることがある。
±秒の補正スライダーを提供し、地図上で写真の位置がリアルタイムに動くプレビューで調整させる。

### 7-3. Android 実装

```kotlin
// 写真選択: Photo Picker API（Android 13+）→ 権限不要でプライバシー良好
val launcher = registerForActivityResult(ActivityResultContracts.PickMultipleVisualMedia()) { uris ->
    uris.forEach { uri -> processPhoto(uri) }
}
launcher.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))

// EXIF読み取り
suspend fun extractPhotoMeta(uri: Uri): PhotoMeta {
    val exif = contentResolver.openInputStream(uri)?.use { ExifInterface(it) }
    val latLng = exif?.latLong               // FloatArray? → [lat, lng]
    val dtRaw  = exif?.getAttribute(ExifInterface.TAG_DATETIME_ORIGINAL)
    return PhotoMeta(uri, latLng?.get(0)?.toDouble(), latLng?.get(1)?.toDouble(),
                     dtRaw?.let { parseExifDateTime(it) })
}

// 軌跡上の位置を補間（タイムスタンプ照合）
fun resolvePosition(meta: PhotoMeta, track: List<TrackPoint>, tzMs: Long, driftMs: Long): LatLng? {
    if (meta.lat != null && meta.lng != null) return LatLng(meta.lat, meta.lng)
    val t = (meta.takenAt ?: return null) - tzMs + driftMs // ローカル→UTC + ドリフト補正
    val idx = track.indexOfLast { it.timestamp <= t }
    if (idx < 0 || idx >= track.size - 1) return null
    val a = track[idx]; val b = track[idx + 1]
    val frac = (t - a.timestamp).toDouble() / (b.timestamp - a.timestamp)
    return LatLng(a.lat + (b.lat - a.lat) * frac, a.lng + (b.lng - a.lng) * frac)
}
```

### 7-4. 表示

- 写真をサムネイルアイコンとして地図マーカーに表示
- 多数の場合は ClusterManager でグループ化
- タップ → 全画面表示 + 撮影日時・場所名（逆ジオコーディング）・紐付け方法の表示
- 再生モードと連動: 再生中に写真を撮影した地点を通過すると写真をポップアップ

### 7-5. データモデル

```kotlin
@Entity(tableName = "photos")
data class PhotoEntity(
    @PrimaryKey val uri: String,
    val lat: Double?,
    val lng: Double?,
    val takenAtUtc: Long?,          // 解決済みのUTCミリ秒
    val matchMethod: String,        // "exif_gps" | "timestamp" | "manual"
    val tzOffsetMinutes: Int,       // 使用したタイムゾーンオフセット（分）
    val driftSeconds: Int,          // クロックドリフト補正値（秒）
    val linkedTrackId: String?,
    val thumbnailPath: String?      // ローカルキャッシュのパス
)
```

### 7-6. ライブラリ

| 用途 | 候補 |
|------|------|
| EXIF読み取り | `androidx.exifinterface:exifinterface:1.3.7` |
| 写真選択 | Photo Picker API（Android 13+） |
| サムネイル表示 | Coil / Glide |
| クラスタリング | Maps SDK ClusterManager |
| 逆ジオコーディング | `Geocoder`（標準） or Nominatim（OSM, 無料） |

### 7-7. 実装優先順位

| 優先度 | 内容 |
|--------|------|
| 1 | Photo Picker → EXIF GPS読み取り → 地図表示 |
| 2 | 撮影日時 + タイムゾーン指定 → 軌跡補間 |
| 3 | サムネイルキャッシュ + クラスタリング |
| 4 | 再生モードとの連動（通過時に写真ポップアップ） |
| 5 | クロックドリフト補正 UI |
| 6 | 逆ジオコーディング（場所名表示） |

---

## 8. 実装優先順位（全体）

| 優先度 | テーマ | 項目 | 理由 |
|--------|--------|------|------|
| 1 | 省電力 | 前面/背面切り替え + PendingIntent化 | 他の全施策の土台 |
| 2 | 省電力 | Significant Motion + Transition API での静止停止 | 効果最大 |
| 3 | 省電力 | 移動手段別の動的間隔 | §2と同時に実装できる |
| 4 | 省電力 | DBバッチ書き込み | 実装コスト小 |
| 5 | 再生 | 時間速度一定・移動速度一定モード | ユーザー向けコア機能 |
| 6 | 写真 | EXIF GPS → 地図表示 | わかりやすい付加価値 |
| 7 | 写真 | タイムゾーン指定 + 日時補間 | GPS無し写真をカバー |
| 8 | 省電力 | Firestore同期のWorkManager化 | 通信コスト削減 |
| 9 | 品質 | 外れ値除去 + 手動補正 | 精度改善 |
| 10 | インポート | GPX / KML読み込み | 外部連携 |
| 11 | 品質 | 道路スナップ（車・自転車） | 見た目改善 |
| 12 | 品質 | 電車区間判別 + 線路スナップ | 技術難度高 |

---

## 9. 未確定事項・今後の調査項目

### アプリ設計の確認が必要なもの

1. **iOS対応するか**  
   → iOS は `PHAsset.location` / `PHAsset.creationDate` を使うため写真連携の実装が別になる。
   　 Flutter/KMP で共通化するかネイティブ2本立てかにより大きく変わる。

2. **地図ライブラリ**  
   → Google Maps SDK / MapLibre / Mapbox のどれを使っているかで ClusterManager の実装が違う。

3. **写真はクラウド同期するか**  
   → 写真本体は数MB/枚。Firestoreに上げるにはStorage併用が必須。
   　 メタデータ（座標・日時）のみ同期 + 本体はローカル限定が現実的か。

4. **オフライン対応が必要か**  
   → 地図タイルのオフラインキャッシュ、道路スナップのローカル処理など。

### 技術調査が必要なもの

5. **バックグラウンド位置の権限フロー**  
   → Android 10+ のバックグラウンド位置権限は2段階でリクエスト必須。Play Store の審査では
   　 「なぜ常に許可が必要か」を具体的に説明する必要がある。

6. **Activity Recognition の電車誤分類の実測**  
   → 実際に IN_VEHICLE がどの程度の頻度で電車に使われるか、ヒューリスティックの
   　 精度を実測する必要あり。

7. **Google フォト API の必要性**  
   → ユーザーがGoogle フォトにしか写真を保存していない場合、Photo Picker 経由なら
   　 Google フォトの写真も選択できる（OAuth審査不要）。

8. **電車の線路データの更新頻度**  
   → OSM データはボランティア更新。新線・廃線への追従方法を決める。

9. **プライバシーポリシーの追記箇所（確定したら追加）**  
   - Activity Recognition 使用（加速度センサーによる行動認識）
   - 写真の EXIF 情報（位置情報・撮影日時）の読み取り
   - 写真データはデバイス外に送信しないこと

---

## 付録: Webプロトタイプ（trajectory.html）について

今回の設計検討と並行して、ブラウザで動く軽量プロトタイプ `trajectory.html` を作成した。
Android実装の前にデータや動作を検証するために使える。

| 機能 | 実装済み |
|------|---------|
| 時間速度一定 / 移動速度一定 モードの再生 | ✅ |
| JSON / GPX / KML / GeoJSON / Google Takeout 読み込み | ✅ |
| 軌跡の手動編集（区間指定・自動補正・点移動・Undo） | ✅ |
| 写真のEXIF GPS / 日時マッチングとオーバーレイ | ✅ |
| タイムゾーン指定・クロックドリフト補正 | ✅ |
| 補正済みJSONのエクスポート | ✅ |

---

*このドキュメントは会話の設計議論をまとめたものです。実装時は各セクションの詳細設計（`docs/battery-optimization.md`、`docs/route-matching-and-import.md`、`docs/photo-gps-matching.md`）も参照してください。*
