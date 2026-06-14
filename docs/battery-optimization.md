# ASHIATO バックグラウンドGPS 省電力化メモ

検討日: 2026-06-13

## 1. 動き出し検知の遅延はなぜ起きるか・どう縮めるか

### 遅延の原因（Activity Recognition の仕組み）

1. **常時監視ではなく周期サンプリング**
   省電力のため、Activity Recognition は加速度センサーを常時読まず、
   一定間隔で起きて数秒分のセンサーデータを取り、分類して、また眠る。
   この「起きる間隔」の分だけ検知が遅れる。

2. **誤検知防止のための確信度待ち**
   ポケットから出した・机が揺れた程度で「移動開始」と判定しないよう、
   複数回のサンプリング窓で一貫して WALKING 等と分類されるまで
   遷移イベント（STILL → WALKING）を発火させない。
   このデバウンスが遅延の主因（典型 30秒〜数分）。

3. **Doze / App Standby による配信遅延**
   画面OFFで端末がDozeに入ると、イベント配信自体が
   メンテナンスウィンドウまで保留されることがある。

### 遅延を縮める方法

| 方法 | 検知遅延 | 消費電力 | 備考 |
|------|---------|---------|------|
| `TYPE_SIGNIFICANT_MOTION` センサー | **数秒** | ほぼゼロ | ハードウェアのセンサーハブ上で動くワンショットセンサー。Dozeでも発火する |
| `TYPE_MOTION_DETECT` (API 24+) | 〜5秒 | ほぼゼロ | 「5秒間動き続けた」で発火。ワンショット |
| Activity **Transition** API | 30秒前後 | 低 | 周期ポーリング版（`requestActivityUpdates`）より速くて省電力。移動手段の分類はこれで |
| ジオフェンス EXIT（最終位置に半径100m） | 1〜3分 | 低 | 保険として併用可。`setNotificationResponsiveness` で応答性を調整 |

### 推奨ハイブリッド構成

```
静止中:  GPS停止。Significant Motion センサーだけ待機（消費ほぼゼロ）
   ↓ 動いた（数秒で発火）
即GPS開始 + lastLocation を先頭点として記録（出発点の取りこぼし防止）
   ↓ 並行して Activity Transition API が移動手段を分類（〜30秒）
分類結果に応じて取得間隔を調整（§3）
   ↓ Transition API が STILL を検知
GPS停止、Significant Motion 待機に戻る
```

ポイント:
- **検知の速さ（Significant Motion）と分類の正確さ（Transition API）を役割分担**させる
- 動き出し直後の空白は `fusedLocationClient.lastLocation` の記録で埋める
- STILL 判定には逆に長めのデバウンス（例: 3分継続でGPS停止）を入れると
  信号待ち・コンビニ立ち寄りで記録が切れない

```kotlin
// 動き出し: ワンショットの Significant Motion
val sensorManager = getSystemService(SensorManager::class.java)
val sigMotion = sensorManager.getDefaultSensor(Sensor.TYPE_SIGNIFICANT_MOTION)
sensorManager.requestTriggerSensor(object : TriggerEventListener() {
    override fun onTrigger(event: TriggerEvent) {
        startLocationUpdates()          // 即GPS開始
        recordLastKnownLocation()       // 出発点を補完
        // ワンショットなので次回静止時に再登録する
    }
}, sigMotion)
```

## 2. バッチ取得の使い分け（バックグラウンド=バッチ / フォアグラウンド=リアルタイム）

方針: **測位は同じ周期で続け、「配信のまとめ方」だけを前後面で切り替える**。

| 状態 | 配信 | 受け取り方 |
|------|------|-----------|
| バックグラウンド | バッチ（例: 最大60秒まとめ） | `PendingIntent`（Broadcast） |
| フォアグラウンド | リアルタイム（遅延0） | `LocationCallback` |

バックグラウンドで `LocationCallback` ではなく **`PendingIntent` 受信にするのが重要**。
プロセスが起きっぱなしにならず、バッチ配信のタイミングだけ起こされるため
CPU・メモリ両方の消費が下がる。

```kotlin
// バックグラウンド用: バッチ + PendingIntent
val bgRequest = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 10_000)
    .setMaxUpdateDelayMillis(60_000)   // 最大60秒まとめて配信
    .build()
fusedClient.requestLocationUpdates(bgRequest, locationPendingIntent)

// フォアグラウンド用: リアルタイム + Callback
val fgRequest = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 3_000)
    .setMaxUpdateDelayMillis(0)        // 即時配信
    .build()
fusedClient.requestLocationUpdates(fgRequest, callback, Looper.getMainLooper())
```

切り替えは `ProcessLifecycleOwner` で行う。**アプリを開いた瞬間に
`flushLocations()` を呼ぶ**と、溜まっているバッチが即吐き出されるので、
地図を開いたとき軌跡が最新まで一気に描かれる。

```kotlin
ProcessLifecycleOwner.get().lifecycle.addObserver(LifecycleEventObserver { _, event ->
    when (event) {
        Lifecycle.Event.ON_START -> {   // 前面へ
            fusedClient.flushLocations()                       // 溜め分を即配信
            fusedClient.removeLocationUpdates(locationPendingIntent)
            fusedClient.requestLocationUpdates(fgRequest, callback, Looper.getMainLooper())
        }
        Lifecycle.Event.ON_STOP -> {    // 背面へ
            fusedClient.removeLocationUpdates(callback)
            fusedClient.requestLocationUpdates(bgRequest, locationPendingIntent)
        }
        else -> {}
    }
})
```

注意:
- バックグラウンド側はフォアグラウンドサービス（`foregroundServiceType="location"`）
  上で動かすこと（Android 10+ のバックグラウンド位置制限のため）
- 前後面で `LocationRequest` を二重登録しないよう、必ず remove → request の順で

## 3. 移動手段に応じた動的設定（やる）

Activity Transition API の分類結果（§1で取得済みのものを流用）で
`LocationRequest` を差し替える。

### パラメータ案

| 移動手段 | 想定速度 | 取得間隔 | 距離フィルタ | 精度 | バッチ上限(背面) |
|---------|---------|---------|------------|------|----------------|
| STILL | 0 | **GPS停止** | – | – | – |
| WALKING | 〜5 km/h | 20秒 | 15 m | HIGH | 120秒 |
| RUNNING | 〜12 km/h | 8秒 | 20 m | HIGH | 60秒 |
| ON_BICYCLE | 〜20 km/h | 5秒 | 25 m | HIGH | 60秒 |
| IN_VEHICLE | 〜60 km/h | 4秒 | 50 m | BALANCED可* | 30秒 |
| UNKNOWN / 分類前 | – | 10秒 | 10 m | HIGH | 60秒 |

\* 車は速度が出るためWiFi測位でも軌跡の形が保てることが多い。
高速道路で曲線が荒れるようなら HIGH に戻す。

考え方: **「1点あたり何m進むか」をだいたい揃える**（上の表は約25〜70m/点）。
徒歩で4秒間隔は無駄打ちだし、車で20秒間隔は軌跡が壊れる。

### 実装スケッチ

```kotlin
val transitions = listOf(
    DetectedActivity.STILL, DetectedActivity.WALKING, DetectedActivity.RUNNING,
    DetectedActivity.ON_BICYCLE, DetectedActivity.IN_VEHICLE
).map {
    ActivityTransition.Builder()
        .setActivityType(it)
        .setActivityTransition(ActivityTransition.ACTIVITY_TRANSITION_ENTER)
        .build()
}
ActivityRecognition.getClient(context)
    .requestActivityTransitionUpdates(ActivityTransitionRequest(transitions), transitionPendingIntent)

// 受信側（BroadcastReceiver）
fun onTransition(activityType: Int) {
    val req = when (activityType) {
        DetectedActivity.STILL      -> null                       // GPS停止 + SigMotion待機へ
        DetectedActivity.WALKING    -> buildRequest(20_000, 15f, 120_000)
        DetectedActivity.RUNNING    -> buildRequest(8_000, 20f, 60_000)
        DetectedActivity.ON_BICYCLE -> buildRequest(5_000, 25f, 60_000)
        DetectedActivity.IN_VEHICLE -> buildRequest(4_000, 50f, 30_000)
        else                        -> buildRequest(10_000, 10f, 60_000)
    }
    swapLocationRequest(req)   // remove → request で差し替え
}

fun buildRequest(intervalMs: Long, minDistM: Float, batchMs: Long) =
    LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, intervalMs)
        .setMinUpdateDistanceMeters(minDistM)
        .setMaxUpdateDelayMillis(if (isForeground) 0 else batchMs)
        .build()
```

注意点:
- 切り替えが頻発しないよう、**同じ手段が続く間は差し替えない**・
  遷移から数十秒は再切り替えしないクールダウンを入れる
- `ACTIVITY_RECOGNITION` 権限（Android 10+）が必要。
  プライバシーポリシーに「加速度センサーによる行動認識を省電力化に使用」を追記すること
- 記録データに移動手段ラベルも保存しておくと、集計（§ポリシー2条の「移動手段などの集計」）にそのまま使える

## 4. その他の省電力アイデア

### 4-1. クラウド同期を充電中・WiFi時に寄せる（効果大）

GPSの次に電池を食うのが**モバイル通信のラジオ起動**。1点記録するたびに
Firestore へ書きに行くとその都度セルラーラジオが起動し、10〜20秒間
高消費状態が続く（Radio Tail と呼ばれる現象）。

**方針**: ローカルDBに溜めて、WorkManager の制約付きジョブでまとめ送り。
前面時（アプリを開いているとき）だけ即時同期すれば UX は変わらない。

```kotlin
// WorkManager で定期バックグラウンド同期（6時間ごと、WiFi + 充電中）
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED) // WiFi のみ
    .setRequiresCharging(true)                     // 充電中のみ
    .build()

val syncWork = PeriodicWorkRequestBuilder<FirestoreSyncWorker>(6, TimeUnit.HOURS)
    .setConstraints(constraints)
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 15, TimeUnit.MINUTES)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "firestore_sync",
    ExistingPeriodicWorkPolicy.KEEP,
    syncWork
)

// Worker 本体
class FirestoreSyncWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        val unsynced = localDb.locationDao().getUnsynced() // 未同期の全レコード
        if (unsynced.isEmpty()) return Result.success()

        return try {
            // Firestore に一括書き込み（バッチ上限500件ずつ分割）
            unsynced.chunked(500).forEach { chunk ->
                val batch = firestore.batch()
                chunk.forEach { pt ->
                    batch.set(firestore.collection("locations").document(), pt.toMap())
                }
                batch.commit().await()
            }
            localDb.locationDao().markSynced(unsynced.map { it.id })
            Result.success()
        } catch (e: Exception) {
            Result.retry() // 次の制約満足タイミングで再試行
        }
    }
}
```

前面に来たときだけ即時同期する切り替え（§2 の ProcessLifecycleOwner に追記）:

```kotlin
Lifecycle.Event.ON_START -> {
    // 前面: 溜まった未同期分を即アップロード（通信があるときだけ）
    if (isNetworkAvailable()) scope.launch { syncToFirestoreNow() }
}
Lifecycle.Event.ON_STOP -> {
    // 背面: WorkManager に任せる（即時同期は止める）
}
```

注意:
- WiFi + 充電中の条件を**両方**つけると同期が数日来ない場合がある。
  許容できるなら条件を緩めて「WiFi または 充電中 かつ 未同期が N 点以上」にする
- `setRequiresStorageNotLow()` も追加するとディスク不足時の書き込みエラーを防げる

---

### 4-2. ローカルDB書き込みのバッチ化（効果中）

1点ごとに `INSERT` するとストレージ I/O のたびにウェイクアップが発生し、
WAL ジャーナルの fsync コストもかかる。**メモリバッファに溜めて
「N点 or 一定時間経過」でトランザクションまとめ書き**にする。

```kotlin
class LocationBuffer(
    private val dao: LocationDao,
    private val maxSize: Int = 30,           // 30点溜まったらフラッシュ
    private val maxAgeMs: Long = 60_000      // 最大60秒に1回は必ずフラッシュ
) {
    private val buffer = mutableListOf<LocationEntity>()
    private var lastFlushAt = System.currentTimeMillis()

    fun add(point: LocationEntity) {
        buffer.add(point)
        val now = System.currentTimeMillis()
        if (buffer.size >= maxSize || now - lastFlushAt >= maxAgeMs) {
            flush()
        }
    }

    fun flush() {
        if (buffer.isEmpty()) return
        val toWrite = buffer.toList()
        buffer.clear()
        lastFlushAt = System.currentTimeMillis()
        // Room のトランザクション一括 INSERT
        CoroutineScope(Dispatchers.IO).launch {
            dao.insertAll(toWrite)
        }
    }
}

// Room DAO 側
@Dao
interface LocationDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(points: List<LocationEntity>)
}
```

フォアグラウンドサービスの `onDestroy` / プロセス終了前に必ず `flush()` を呼ぶこと。
クラッシュ時の取りこぼしが気になるなら `maxSize` を小さく（10点程度）にする。

---

### 4-3. バッテリーセーバー連動（効果中・体験配慮）

端末のバッテリー残量・省電力モードを監視し、自動でロケーション取得間隔を調整する。
**ユーザーが気づかないうちに精度が落ちる**のは避けたいので、
通知で「省電力モードに切り替えます」と伝えるか、設定で3段階から選べるようにするのがベスト。

#### 仕組み1: バッテリー残量による自動切り替え

```kotlin
// BroadcastReceiver でバッテリーイベントを受信
class BatteryReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val level  = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
        val scale  = intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
        val pct    = if (scale > 0) level * 100 / scale else return

        val mode = when {
            pct <= 15 -> TrackingMode.POWER_SAVE   // 間隔2倍 or 記録停止通知
            pct <= 30 -> TrackingMode.BALANCED
            else      -> TrackingMode.NORMAL
        }
        LocationSettingsManager.applyMode(mode)
    }
}

// AndroidManifest に登録
// <action android:name="android.intent.action.BATTERY_CHANGED"/>
```

#### 仕組み2: システムの省電力モード（Battery Saver）連動

```kotlin
val powerManager = getSystemService(PowerManager::class.java)

// 現在の状態を確認
val isSaving = powerManager.isPowerSaveMode

// 変化を検知
registerReceiver(object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val saving = powerManager.isPowerSaveMode
        LocationSettingsManager.applyMode(
            if (saving) TrackingMode.POWER_SAVE else TrackingMode.NORMAL
        )
    }
}, IntentFilter(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED))
```

#### 3段階モードの定義例

```kotlin
enum class TrackingMode {
    POWER_SAVE, // 省電力: 間隔 2倍、BALANCED精度
    BALANCED,   // 標準:   §3 の動的間隔そのまま
    NORMAL      // 高精度: HIGH精度、間隔短め
}

object LocationSettingsManager {
    fun applyMode(mode: TrackingMode) {
        val multiplier = when (mode) {
            TrackingMode.POWER_SAVE -> 2.0f
            TrackingMode.BALANCED   -> 1.0f
            TrackingMode.NORMAL     -> 0.75f
        }
        val priority = when (mode) {
            TrackingMode.POWER_SAVE -> Priority.PRIORITY_BALANCED_POWER_ACCURACY
            else                    -> Priority.PRIORITY_HIGH_ACCURACY
        }
        // 現在の ActivityType に応じた間隔 × multiplier で LocationRequest を再構築
        currentActivityType?.let { rebuildRequest(it, multiplier, priority) }
    }
}
```

ユーザー設定で3段階を固定することもでき、その場合 `multiplier` をユーザー選択で固定すれば同じ仕組みが使い回せる。

---

### 4-4. 後処理での軌跡補正（効果中）

取得間隔を広げると点が荒くなるが、**表示・集計時の後処理で品質を保てる**。
「点を増やして綺麗にする」から「少ない点を賢く描く」への発想転換。

#### Douglas-Peucker 簡略化

地図のズームレベルに合わせて表示点数を間引く。拡大表示時は元の点を使い、
縮小表示では簡略化した点列を使うことで描画負荷も下がる。

```kotlin
// ε = 許容誤差（メートル換算）。ズームに合わせて変える
fun douglasPeucker(points: List<LatLng>, epsilon: Double): List<LatLng> {
    if (points.size < 3) return points
    var maxDist = 0.0; var maxIdx = 0
    val end = points.last()
    for (i in 1 until points.size - 1) {
        val d = perpendicularDistance(points[i], points.first(), end)
        if (d > maxDist) { maxDist = d; maxIdx = i }
    }
    return if (maxDist > epsilon) {
        douglasPeucker(points.subList(0, maxIdx + 1), epsilon) +
        douglasPeucker(points.subList(maxIdx, points.size), epsilon).drop(1)
    } else {
        listOf(points.first(), points.last())
    }
}

fun perpendicularDistance(p: LatLng, a: LatLng, b: LatLng): Double {
    // 線分 a-b からの垂直距離（度単位→メートル近似）
    val dx = b.longitude - a.longitude; val dy = b.latitude - a.latitude
    val len2 = dx*dx + dy*dy
    if (len2 == 0.0) return haversineMeters(p, a)
    val t = ((p.longitude - a.longitude)*dx + (p.latitude - a.latitude)*dy) / len2
    val px = a.longitude + t*dx; val py = a.latitude + t*dy
    return haversineMeters(p, LatLng(py, px))
}
```

#### 速度・精度フィルタ（外れ値除去）

GPS 誤差で「一瞬100m飛んだ」ような異常点を除去する。

```kotlin
fun filterOutliers(points: List<LocationEntity>): List<LocationEntity> {
    val result = mutableListOf<LocationEntity>()
    for (i in points.indices) {
        if (i == 0) { result.add(points[i]); continue }
        val prev = result.last()
        val dist = haversineMeters(prev.toLatLng(), points[i].toLatLng())
        val dt   = (points[i].timestamp - prev.timestamp) / 1000.0 // 秒
        val impliedSpeed = if (dt > 0) dist / dt else 0.0          // m/s

        // 時速200km超（55 m/s）は GPS 誤差とみなして除外
        if (impliedSpeed < 55.0 || points[i].accuracy < 30f) {
            result.add(points[i])
        }
    }
    return result
}
```

#### スナップ補正（Maps Roads API / Google Maps SDK）

点が道路・歩道から外れている場合、最寄りの道路に吸着させる後処理。
車・自転車の区間で特に効果的。

```
Google Roads API の snapToRoads エンドポイントに点列を送ると
補正済みの点列が返ってくる（有料 API、課金注意）
```

実装方針:
- 車・自転車判定の区間だけスナップをかける（徒歩は道路外を歩く場合がある）
- ローカルで完結させたい場合は OSRM の map-matching API（OSS）を自前ホストする選択肢もある
- クラウド同期のタイミング（§4-1 の SyncWorker）と合わせてバックグラウンドで処理し、
  補正済み座標を別カラムに保存して表示時に使い分けるのが無難

---

### 4-5. やっても効果が薄いもの（参考）

- **距離フィルタ単体**: GPSチップ自体は動き続けるので電池への効果は小さい。
  DB・通信コストの節約にはなる
- `setGranularity` / `setWaitForAccurateLocation`: 品質調整であり消費はほぼ変わらない
- **WakeLock の見直し**: FLP + PendingIntent 構成（§2）にすれば自前 WakeLock は不要になる

---

## 実装の優先順位（提案）

| 優先度 | 内容 | 主な効果 |
|--------|------|---------|
| 1 | **§2 前面/背面切り替え + PendingIntent 化** | 構造の土台。先にやらないと他が乗らない |
| 2 | **§1 Significant Motion + Transition API** | 静止中 GPS ゼロ。効果最大 |
| 3 | **§3 移動手段別の動的間隔** | §1 の Transition 結果を流用するだけ |
| 4 | **§4-2 ローカルDB バッチ書き込み** | 実装コスト小・副作用なし |
| 5 | **§4-1 Firestore 同期を WorkManager 化** | 通信系の節約 |
| 6 | **§4-3 バッテリーセーバー連動** | ユーザー体験配慮が必要。後半で |
| 7 | **§4-4 後処理補正** | 取得間隔を広げた後の品質担保として |
