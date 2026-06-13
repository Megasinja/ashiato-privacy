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
GPSと並んで電池を食うのが**モバイル通信のラジオ起動**。
Firestoreへの同期を記録のたびに行わず、ローカルDBに貯めて
WorkManager の制約付きジョブでまとめ送りする。

```kotlin
val syncWork = PeriodicWorkRequestBuilder<SyncWorker>(6, TimeUnit.HOURS)
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.UNMETERED)  // WiFi時
        .setRequiresCharging(true)                      // 充電中
        .build())
    .build()
```
リアルタイム同期が欲しいのはアプリを開いている時だけなので、
前面時のみ即時同期に切り替えれば体験は変わらない。

### 4-2. ローカルDB書き込みもバッチ化（効果中）
1点ごとに INSERT せず、メモリに数十点貯めてトランザクションでまとめ書き。
ストレージI/Oのウェイクアップが減る。クラッシュ対策に
「N点 or 60秒ごと」のフラッシュにする。

### 4-3. バッテリーセーバー連動（効果中・体験配慮）
`PowerManager.isPowerSaveMode` / バッテリー残量を見て、
低残量時は自動で間隔を2倍に伸ばす or 記録一時停止を提案する通知を出す。
ユーザー設定に「省電力モード（粗め）/ 標準 / 高精度」の3段階を
用意しておくと、ユーザー側でトレードオフを選べる。

### 4-4. 後処理で軌跡を補正し、生データの密度を下げる（効果中）
取得密度を下げると軌跡が粗くなる問題は、表示時の処理である程度補える:
- Douglas-Peucker 簡略化はすでに点が多い場合の表示高速化に
- 徒歩/車の区間はスナップ補正（道路・歩道に吸着）で粗い点列でも見栄えが保てる
「取得を増やして綺麗にする」のではなく「少ない点を賢く描く」方向。

### 4-5. やっても効果が薄いもの（参考）
- 距離フィルタ単体: 測位自体は続くので電池への効果は小さい（§前回の議論どおり）
- `setGranularity` / `setWaitForAccurateLocation`: 品質調整であり消費はほぼ変わらない
- WakeLock の見直し: FLP + PendingIntent 構成にすれば自前WakeLockは不要になる

## 実装の優先順位（提案）

1. **§2 前面/背面切り替え + PendingIntent化** — 構造の土台。先にやる
2. **§1 Significant Motion + Transition API による静止時GPS停止** — 効果最大
3. **§3 移動手段別の動的間隔** — §1のTransition結果を流用するだけなので追加コスト小
4. **§4-1 同期のWorkManager化** — 通信系の節約
5. §4-2, 4-3 は仕上げ
