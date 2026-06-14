# ASHIATO 写真×GPS軌跡 突合機能 設計メモ

検討日: 2026-06-14

---

## 1. やりたいこと

GPS 軌跡に沿って「そのとき撮った写真」を地図上に重ねて表示する。
「あのとき何処にいたか」と「そこで撮った写真」が一気に見られる体験。

紐付けの根拠は2種類あり、どちらか使えるほうを優先する:

| 方法 | 精度 | 使える条件 |
|------|------|-----------|
| **EXIF GPS** | 〜5 m | カメラアプリが位置情報を写真に埋め込んでいる |
| **撮影日時** | 〜30 秒 | 写真に撮影日時があり、GPS軌跡の時刻と合わせられる |

---

## 2. 紐付けの詳細設計

### 2-1. EXIF GPS 方式

写真の EXIF タグには GPSLatitude / GPSLongitude が入っている場合がある。
これをそのまま地図座標として使う。

- メリット: 正確、軌跡データが無くても使える
- デメリット: カメラ設定で位置情報 OFF にしていると空。SNS 等で剥がされた写真は使えない
- 注意: EXIF の GPS は WGS-84 座標系（Google マップと同じ）なので変換不要

```
優先度 1: EXIF の GPSLatitude / GPSLongitude が存在する → それを採用
```

### 2-2. 撮影日時 → 軌跡補間 方式

EXIF の `DateTimeOriginal`（撮影日時）を読み、GPS 軌跡上でその時刻に
いた位置を線形補間して求める。

```
優先度 2: GPSLatitude が無い → DateTimeOriginal を読む
         → 軌跡の timestamp で二分探索 → 前後点を線形補間 → 座標確定
```

#### タイムゾーン問題（最重要）

EXIF の `DateTimeOriginal` は **ローカル時刻・タイムゾーン情報なし**。
一方、GPS 軌跡のタイムスタンプは通常 UTC または端末のローカル時刻。

| データ | 形式 | 例 |
|--------|------|-----|
| EXIF DateTimeOriginal | `YYYY:MM:DD HH:MM:SS`（ローカル時刻）| `2024:06:01 09:15:30` |
| GPS タイムスタンプ (NMEA/GPX) | UTC | `2024-06-01T00:15:30Z` |
| Androidアプリ記録 | 端末のUNIXミリ秒 | `1717201530000` |

対処方法:
1. **自動推定**: 軌跡の先頭・末尾のUTC時刻とEXIF時刻を比較し、
   オフセットを自動推定（例: 差が9時間ならJST）
2. **ユーザー指定**: UTC+9（JST）などを設定画面で指定させる
3. **EXIF OffsetTimeOriginal** タグを使う（対応カメラは少ない）

日本国内での利用がメインならデフォルト JST（UTC+9）で問題ない場合が多い。

#### クロックドリフト問題

カメラの時計が GPS 時刻と数秒〜数十秒ずれていることがある。
→ ユーザーが「写真の時刻を ±n 秒補正」できる調整機能を用意する。

---

## 3. 必要調査事項

### Android 側

| 調査項目 | 内容 | 参考 |
|---------|------|------|
| **MediaStore 権限** | Android 13+ は `READ_MEDIA_IMAGES`、それ以前は `READ_EXTERNAL_STORAGE` | AndroidManifest の `uses-permission` |
| **ExifInterface** | `androidx.exifinterface:exifinterface` で EXIF を読む。`Uri` から `InputStream` を開いて渡す | Jetpack ライブラリ |
| **ContentResolver クエリ** | `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` で端末内写真一覧を取得 | Cursor API |
| **サムネイル取得** | `ContentResolver.loadThumbnail(uri, size, ...)` (API 29+) / `MediaStore.Images.Thumbnails` (旧) | 大量写真の表示に必須 |
| **写真の時刻** | MediaStore の `DATE_TAKEN` vs EXIF `DateTimeOriginal` の違い（前者はミリ秒UTC）| 精度差あり |
| **Scoped Storage** | Android 10+ は自分のアプリ以外の写真へのアクセスに権限が必要 | Scoped Storage ガイド |
| **フォトピッカー** | `Photo Picker API`（Android 13+ / Pixel backport）: 権限不要でユーザーが選んだ写真だけアクセス可 | プライバシー観点で推奨 |

### クロスプラットフォーム（iOS）

| 調査項目 | 内容 |
|---------|------|
| `PHAsset.location` | `CLLocation?` として GPS 座標を返す（EXIF 読み不要） |
| `PHAsset.creationDate` | `Date?` として撮影日時（UTC）を返す |
| `PHImageManager.requestImage` | サムネイル取得 |
| `PHPhotoLibrary.requestAuthorization` | 権限要求。iOS 14+ は `limited` 選択も |

---

## 4. Android 実装手順

### Step 1: 依存関係を追加

```kotlin
// app/build.gradle
dependencies {
    implementation("androidx.exifinterface:exifinterface:1.3.7")
    // (Glide or Coil for image loading / thumbnail)
    implementation("io.coil-kt:coil:2.6.0")
}
```

### Step 2: 権限の宣言と要求

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES"
    android:minSdkVersion="33"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32"/>
```

```kotlin
// Android 13+ はフォトピッカーを優先（権限要求不要）
val photoPickerLauncher = registerForActivityResult(
    ActivityResultContracts.PickMultipleVisualMedia()
) { uris -> processPhotos(uris) }

// フォトピッカー起動
photoPickerLauncher.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
```

### Step 3: 写真から GPS / 日時を抽出

```kotlin
data class PhotoMeta(
    val uri: Uri,
    val lat: Double?,    // EXIF GPS（null なら日時で紐付け）
    val lng: Double?,
    val takenAt: Long?,  // DateTimeOriginal → ミリ秒（ローカル時刻）
    val dateTaken: Long? // MediaStore DATE_TAKEN（UTC ミリ秒）
)

suspend fun extractPhotoMeta(uri: Uri, resolver: ContentResolver): PhotoMeta {
    val exif = resolver.openInputStream(uri)?.use { ExifInterface(it) }

    // GPS座標（ExifInterface が度への変換をやってくれる）
    val latLng = exif?.latLong // FloatArray?
    val lat = latLng?.get(0)?.toDouble()
    val lng = latLng?.get(1)?.toDouble()

    // 撮影日時 (EXIF: "2024:06:01 09:15:30")
    val dtRaw = exif?.getAttribute(ExifInterface.TAG_DATETIME_ORIGINAL)
    val takenAt = dtRaw?.let { parseExifDateTime(it) }

    // MediaStore DATE_TAKEN（精度はやや落ちるが UTC で扱いやすい）
    val dateTaken = resolver.query(uri, arrayOf(MediaStore.Images.Media.DATE_TAKEN), null, null, null)
        ?.use { c -> if (c.moveToFirst()) c.getLong(0) else null }

    return PhotoMeta(uri, lat, lng, takenAt, dateTaken)
}

// "2024:06:01 09:15:30" → エポックミリ秒（ローカル時刻として解釈）
fun parseExifDateTime(s: String): Long? {
    return runCatching {
        val (date, time) = s.split(" ")
        val (y, mo, d) = date.split(":").map { it.toInt() }
        val (h, mi, sec) = time.split(":").map { it.toInt() }
        Calendar.getInstance().apply {
            set(y, mo - 1, d, h, mi, sec); set(Calendar.MILLISECOND, 0)
        }.timeInMillis
    }.getOrNull()
}
```

### Step 4: GPS 軌跡上の座標に紐付ける

```kotlin
data class TrackPoint(val lat: Double, val lng: Double, val timestamp: Long)

fun resolvePhotoPosition(
    meta: PhotoMeta,
    track: List<TrackPoint>,
    tzOffsetMs: Long = 9 * 3600 * 1000L // デフォルト JST
): LatLng? {
    // 優先1: EXIFにGPS座標がある
    if (meta.lat != null && meta.lng != null) return LatLng(meta.lat, meta.lng)

    // 優先2: 撮影日時 → 軌跡補間
    val t = (meta.takenAt?.minus(tzOffsetMs))     // ローカル→UTC
           ?: meta.dateTaken                        // MediaStore は既にUTC
           ?: return null

    if (t < track.first().timestamp || t > track.last().timestamp) return null

    val idx = track.indexOfLast { it.timestamp <= t }
    if (idx < 0 || idx >= track.size - 1) return null

    val a = track[idx]; val b = track[idx + 1]
    val frac = (t - a.timestamp).toDouble() / (b.timestamp - a.timestamp)
    return LatLng(a.lat + (b.lat - a.lat) * frac, a.lng + (b.lng - a.lng) * frac)
}
```

### Step 5: 地図上に表示（Marker + サムネイル）

```kotlin
// Compose + Google Maps SDK の場合
@Composable
fun PhotoMarker(photo: PhotoMeta, position: LatLng, onTap: () -> Unit) {
    val bitmap = remember { loadThumbnail(photo.uri, 80) } // 80×80px
    MarkerComposable(state = MarkerState(position)) {
        AsyncImage(
            model = photo.uri,
            contentDescription = null,
            modifier = Modifier.size(48.dp).clip(RoundedCornerShape(4.dp))
                .border(2.dp, Color.White, RoundedCornerShape(4.dp))
                .clickable { onTap() }
        )
    }
}

// マーカー多数によるパフォーマンス対策: ズームレベルに応じてクラスタリング
// → Maps SDK for Android の ClusterManager を使う
```

### Step 6: タップ時の詳細表示

写真タップ → BottomSheet で
- 元写真（フル）表示
- 撮影日時、場所名（逆ジオコーディング）
- 地図ミニビュー（この点を中心に小さな地図）
- 「この写真を削除」「軌跡との紐付けを修正」

---

## 5. データモデル

```kotlin
// Room テーブル
@Entity(tableName = "photos")
data class PhotoEntity(
    @PrimaryKey val uri: String,
    val lat: Double?,
    val lng: Double?,
    val takenAt: Long?,          // 解決済みの UTC ミリ秒
    val matchMethod: String,     // "exif_gps" | "timestamp" | "manual"
    val tzOffsetUsed: Int,       // 使用したタイムゾーンオフセット（分）
    val linkedTrackId: String?,  // どの軌跡に紐付いているか
    val thumbnailPath: String?   // キャッシュしたサムネイルのパス
)
```

写真と軌跡の関係は N:1（写真が属する軌跡は1つ）。
ただし1日に複数の軌跡がある場合は最も近い軌跡セグメントに自動で割り当てる。

---

## 6. UI/UX 設計

### 地図ビュー（写真オーバーレイ）

```
[📷 写真を表示] トグルボタン → ON で写真マーカーをオーバーレイ
ズームイン  → 個別写真のサムネイルが見える
ズームアウト → クラスタリング（「3枚」などのバッジ）
写真タップ  → 詳細ボトムシート
```

### タイムライン/スライドショー

軌跡の再生（trajectory.html 的な機能）と写真を組み合わせ、
「移動しながら写真が現れる」スライドショーモード:
- 再生中に写真を撮影した地点を通過すると、写真がポップアップ
- 右スワイプで前の写真、左スワイプで次の写真

### 時刻補正 UI

```
[⏱ 写真の時刻補正]
タイムゾーン: [UTC+9 ▾]
補正: [-30秒 ─────●───── +30秒]  (クロックドリフト用)
      プレビュー: 写真Aが地図上の▲に表示
```

---

## 7. 技術スタック・ライブラリ

| 用途 | 採用候補 | 備考 |
|------|---------|------|
| EXIF 読み取り (Android) | `androidx.exifinterface` | Jetpack 公式。GPS・日時タグ対応 |
| 写真選択 | `Photo Picker API` (Android 13+) | 権限不要・プライバシー良好 |
| サムネイル表示 | `Coil` or `Glide` | Uri から非同期読み込み |
| 地図クラスタリング | `Maps SDK ClusterManager` | 多数マーカーのまとめ表示 |
| 逆ジオコーディング | `Geocoder` (Android標準) or `Nominatim (OSM)` | 「渋谷区神南」などの地名取得 |
| EXIF 読み取り (Web) | `exifr` (npm / CDN) | 3KB gzip、GPS + 日時対応 |
| 写真クラスタリング (Web) | `Leaflet.markercluster` | trajectory.html に追加可能 |

---

## 8. プライバシー・権限

- 写真はデバイス内または自分の Firestore に限定。第三者送信しない
- EXIF GPS 情報は他人の写真（共有写真）では不用意に使わない
- Google フォトや SNS からインポートする場合は各サービスの API 規約を確認
- プライバシーポリシーに追記:
  「写真の EXIF 情報（位置情報・撮影日時）をGPS軌跡との紐付けに使用します。
   写真データはデバイス内および利用者本人のクラウドストレージ外に送信しません」
- Android Photo Picker を使うと「どの写真を選んだか」のみアクセスできるため、
  ライブラリ全体を読まれるリスクがなく、ユーザーへの説明が楽

---

## 9. 実装の優先順位

| 優先度 | 内容 | コスト | 効果 |
|--------|------|--------|------|
| 1 | **Photo Picker → EXIF GPS 読み取り → 地図表示** | 小 | GPS埋め込みがある写真は即対応 |
| 2 | **撮影日時 + タイムゾーン設定 → 軌跡補間** | 中 | GPS無し写真も対応 |
| 3 | **サムネイルキャッシュ + クラスタリング** | 中 | 写真が多くても快適 |
| 4 | **タイムライン/スライドショーモード** | 大 | 体験の目玉機能 |
| 5 | **時刻補正 UI（ドリフト・タイムゾーン）** | 小 | 細かい精度向上 |
| 6 | **逆ジオコーディング** | 小 | 「どこで撮ったか」の文字表示 |

---

## 10. 調査が必要な未確定事項

1. **アプリの対応 OS は Android 単体か iOS も含むか**
   → iOS は `PHAsset.location` で GPS が取れるため実装が違う

2. **写真はアプリ内で見るだけか、外部と共有するか**
   → 共有する場合はEXIF除去の選択肢を用意するかどうか

3. **クラウド同期の対象にするか**
   → 写真本体は重い（数MB/枚）。サムネイル + メタデータのみ Firestore に上げ、
     本体はローカル限定が現実的

4. **Google フォト API との連携は必要か**
   → Google Photos Library API で Takeout 不要でアクセスできるが、OAuth・審査が必要

5. **地図ライブラリは何を使っているか**
   → MapLibre / Mapbox / Google Maps SDK で `MarkerClusterer` の使い方が異なる
