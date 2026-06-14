# ASHIATO ルート画像 → 座標変換（ジオリファレンス／ベクトル化）設計メモ

調査日: 2026-06-14

> このドキュメントは Web 実調査に基づく。各APIの規約・料金は変動するため、
> 実装前に公式の最新条件を必ず確認すること。出典は末尾。

---

## 0. 用語の整理（重要）

会話で「ルート検索で出た**画像**を、**座標情報（緯度経度の列）に変換**したい。
そこをジオコーディング／逆ジオコーディングと呼んでいた」という話があった。

この「画像 → 座標」は、正確には**ジオコーディングではない**。正しい用語は:

| ユーザーの言い方 | 正しい用語 | 意味 |
|----------------|-----------|------|
| 画像→座標の「ジオコーディング」 | **ジオリファレンス**（georeferencing） | 画像のピクセル位置 ↔ 地理座標（緯度経度）の対応づけ |
| 〃 | **ベクトル化／デジタイズ**（vectorization） | ラスター画像中の「線」を座標の列（polyline）に起こす |
| （本来の）ジオコーディング | geocoding | **住所・地名 ⇄ 座標** の変換（別物） |

つまり「ルート画像 → 座標」は **ジオリファレンス＋ベクトル化** の組み合わせ。
住所⇄座標の geocoding とは無関係なので、設計・調査もそちらの技術で行う。

---

## 1. 結論（先に要点）：まず API 応答の座標を使う

**多くの場合、画像から座標を起こす必要はない。** ルート検索の結果は、
地図に描かれる「画像」とは別に、**応答の中に座標データが最初から入っている**。
画像はその座標を描画した結果にすぎない。

| ソース | 座標の取得方法 | 形式 | 保存可否 |
|--------|--------------|------|---------|
| **Google Directions API** | `routes[].overview_polyline.points` を**デコード** | エンコード済みポリライン → 緯度経度列 | ❌ 恒久不可（30日キャッシュのみ） |
| **OpenRouteService** | `/v2/directions` のレスポンス | **GeoJSON 座標列**（そのまま） | ✅ ODbL（帰属＋ShareAlike） |
| **Yahoo YOLP** | 「経路地図API（画像）」ではなく**「経路探索」**を使う | JS地図上のジオメトリ | 要Yahoo規約確認 |
| **駅すぱあと / NAVITIME** | 経路探索APIのレスポンス | 座標列 | 契約による |

### ポイント
- **Google**: `overview_polyline` は Encoded Polyline Algorithm で圧縮された文字列。
  `google.maps.geometry.encoding.decodePath()` や各言語のデコーダで緯度経度列に戻せる。
  → **画像処理は不要**。ただし経路ジオメトリは**恒久保存できない**ので、
    「基準ルートとして永続保存」用途には使えない（`routing-api-options.md` §1-1）。
    **座標にデコードしても保存制限は外れない**（§6 で詳述）。
- **ORS**: 最初から GeoJSON 座標列。ODbL を守れば保存可。**基準ルートの本命**。
- **Yahoo**: 「経路地図API」は経路を**画像**で返すので座標が要るなら不向き。
  代わりに「経路探索」を使えばジオメトリが取れる。

> したがって **画像 → 座標の処理は「最後の手段」**。API が使えるなら応答の
> polyline / GeoJSON を直接使うのが正攻法で、精度も完璧、実装も軽い。

---

## 2. 本当に「画像しか無い」場合のパイプライン

API が無く、**ルートが描かれた画像だけ**がある場合（紙地図、掲示板の写真、
画像でしか結果を返さないサービス等）に初めて、画像から座標を起こす処理が要る。
研究例として **RouteExtract**（紙地図→GPX のモジュラーパイプライン, arXiv 2509.11674）がある。

```
入力: ルート線が描かれた地図画像
  │
  ├ Step 1  ジオリファレンス（ピクセル↔緯度経度の変換を確立）
  ├ Step 2  線抽出（ルート線を色でマスク → 二値画像）
  ├ Step 3  スケルトン化（線を1px幅の中心線に細線化）
  ├ Step 4  トレース（ピクセルをグラフ化し始点→終点に順序づけ）
  ├ Step 5  単純化（Douglas-Peucker で点を間引く）
  ├ Step 6  投影（各ピクセル(x,y) → (lat,lng)）
  └ Step 7  出力（GPX / GeoJSON）
```

### Step 1: ジオリファレンス

ピクセル座標と地理座標の対応づけ。2通りある。

- **(a) 自分が生成した静的地図（中心・ズーム・画像サイズが既知）**
  → **Web Mercator 逆変換で厳密に解ける**（§3）。誤差は実質ゼロ。
    例: Static Map API を `center, zoom, size` 指定で取得した画像。
- **(b) 任意のスクショ・紙地図（パラメータ不明）**
  → **Ground Control Points (GCP)** を使う。画像内の既知地点（交差点・駅・ランドマーク）
    を3点以上、緯度経度と対応づけてアフィン／射影変換を推定する。誤差は GCP の精度次第。

### Step 2: 線抽出（色マスク）

ルート線は通常、地図の他要素と異なる**目立つ単色**（例: 青や赤）で描かれる。
HSV 色空間でしきい値処理して、その色のピクセルだけを抽出 → 二値マスク。

- OpenCV `cv2.inRange(hsv, lower, upper)` で色マスク。
- アンチエイリアスや半透明線は範囲を少し広めに取る。

### Step 3: スケルトン化（細線化）

太い線を1ピクセル幅の中心線にする。

- `skimage.morphology.skeletonize`（scikit-image）
- または LingDong `skeleton-tracing`（二値画像→ポリライン群を直接出す軽量アルゴリズム）

### Step 4: トレース（順序づけ）

スケルトンのピクセルを**8連結グラフ**にして、端点（次数1のノード）から
反対の端点まで経路を辿り、座標の順序を決める。

- NetworkX で 8連結のピクセルグラフを構築 → 端点検出 → 最長路/最短路でトレース。
- 交差・分岐があるルートは、ここで順序が乱れやすい（§5 失敗モード）。

### Step 5: 単純化

トレース結果はピクセル数ぶんの点になるので、**Douglas-Peucker** で間引く。
（アルゴリズムの説明は `battery-optimization.md` §4-2 に既出。再掲しない。）

### Step 6: 投影（ピクセル → 緯度経度）

Step 1 で確立した変換を各点に適用してピクセル(x,y)→(lat,lng)に変換。

### Step 7: 出力

GPX または GeoJSON で書き出す。
→ 生成物は `trajectory.html` の 📁 から読み込んで元地図と重ね、ズレを目視検証できる。

---

## 3. Web Mercator 逆変換の数式（静的地図・既知パラメータ）

Google/OSM 等の地図は **Web Mercator（EPSG:3857）**。
中心 `(latC, lngC)`、ズーム `z`、画像サイズ `W×H`（px）が分かっていれば、
画像中心を基準に各ピクセルを厳密に緯度経度へ戻せる。

### 緯度経度 → ワールドピクセル（順変換）

タイルサイズ `TILE=256` とすると、ズーム z のワールド全体は `256·2^z` px。

```
scale = 256 * 2^z
worldX(lng) = (lng + 180) / 360 * scale
worldY(lat) = (1 - ln( tan(lat·π/180) + 1/cos(lat·π/180) ) / π) / 2 * scale
```

### ワールドピクセル → 緯度経度（逆変換）

```
lng = worldX / scale * 360 - 180
n   = π - 2π · worldY / scale
lat = (180/π) · atan( 0.5 · (e^n - e^-n) )      // = atan(sinh(n))
```

### 画像内ピクセル(px,py) → 緯度経度

画像中心が地図中心 `(latC,lngC)` に対応するので:

```
worldCenterX = worldX(lngC)
worldCenterY = worldY(latC)
worldX = worldCenterX + (px - W/2)
worldY = worldCenterY + (py - H/2)
→ 上の逆変換で (lat, lng) を得る
```

> これは「ジオコーディング」ではなく**投影の逆計算**。中心・ズーム・サイズが既知なら
> GCP すら不要で、ピクセル誤差≒地上数m（ズーム次第）に収まる。

---

## 4. 別アプローチ：インタラクティブ地図なら DOM/JS から取る

ルートが**画像ではなくJS地図（Leaflet/Maps JS等）として描画**されている場合、
スクショを撮る必要はない。**描画に使われた座標がページ内にすでにある**。

- Leaflet: `polyline.getLatLngs()` で座標列を取得。
- Google Maps JS: ルートの `overview_path` / `LatLng[]` を読む。
- → これも「画像→座標」ではなく「描画データを読むだけ」。最も確実。

---

## 5. 精度と失敗モード

| 問題 | 原因 | 対策 |
|------|------|------|
| 線が途切れる | ラベル文字・他の線がルート線に重なる / 色が被る | 色マスクの調整、モルフォロジー膨張で隙間を埋める |
| 順序が乱れる | 交差・ループ・分岐 | グラフトレースで端点起点に辿る。分岐は手動補正 |
| 全体がずれる | ジオリファレンス誤差（GCP不足・低精度） | GCPを増やす／静的地図の既知パラメータ方式に切替 |
| ガタつく | スケルトンのジグザグ | Douglas-Peucker＋平滑化（`trajectory.html` の自動補正と同系） |

期待精度の目安:
- **静的地図（中心・ズーム既知）**: 数px = 地上数m。実用十分。
- **任意スクショ＋GCP**: GCP精度に依存、十m〜数十m。
- **紙地図の写真**: 撮影歪み・紙のしわで誤差大。GCPを多めに。

---

## 6. 法務・規約の注意（重要）

- **Google/Yahoo の描画ルート画像から幾何を抽出して恒久保存する**のは、
  各社の保存制限（Googleは経路の恒久保存不可）を**回避する目的だと規約上グレー**。
  「画像にして座標を起こせば保存OK」にはならない。**推奨しない。**

### ⚠️ 「Googleの座標を自前ロジックで永久化」はできない

> よくある誤解: 「Googleの30日制限は画像/キャッシュの話。座標さえ取れれば
> 自分のDBに入れて永久保存できるのでは？」

これは**できない**。理由:

- Google の制限は**形式（画像 / エンコードされたポリライン / デコード済み緯度経度）に
  依らない**。`overview_polyline` をデコードして得た座標も "Google Maps Content" であり、
  **30日を超える保存・再配布は規約違反**。place_id だけが無期限保存可の例外。
- 「自前ロジックで永久化」は、まさに規約が禁じている**保存制限の回避**にあたる。
- リスク: API キーの利用停止・アカウント停止、契約違反。商用アプリでは現実的に取れない。
- 画像経由でも座標直取得でも、**Google 由来である限り結論は同じ**。

**したがって「永久保存して基準ルートに使う」なら、保存可能なソースから取る**:

| 目的 | 使うべきソース | 保存 |
|------|--------------|------|
| 車・徒歩・自転車のルートを保存 | **OpenRouteService**（OSM/ODbL） | ✅ 帰属＋ShareAlike |
| 電車・バスのルートを保存（日本） | **駅すぱあと API / NAVITIME API** | ✅ 契約に基づく |
| その場で表示するだけ（保存しない） | Google Directions | △ 表示のみ・30日キャッシュまで |

→ ORS なら応答が最初から GeoJSON 座標列なので、**画像処理も Google も不要**で
   永久保存・スナップ基準にそのまま使える。これが本筋。

### 正当なユースケース（画像→座標が妥当な場面）

- 紙地図・登山道の掲示板地図・観光案内図など、**デジタル元データが存在しない**ものの
  デジタイズ（RouteExtract の原典がまさにこれ）。
- 自分が権利を持つ／ライセンス上問題ない画像。

### 保存して使いたいなら（再掲）

画像処理ではなく**保存可能なソースから座標を取る** — OpenRouteService（ODbL）、
駅すぱあと / NAVITIME（契約）。詳細は `routing-api-options.md`。

---

## 7. ツール早見表

| 用途 | ツール | 備考 |
|------|--------|------|
| 画像処理全般・色マスク | **OpenCV** | `inRange`, モルフォロジー, 輪郭 |
| スケルトン化 | **scikit-image** `skeletonize` | 1px中心線 |
| スケルトン→ポリライン | **skeleton-tracing**（LingDong） | 二値画像から直接polyline群 |
| ピクセルグラフ・トレース | **NetworkX** | 8連結グラフ、端点・分岐検出 |
| 汎用ラスター→ベクター | **VTracer** | 画像全体のベクター化（線特化ではない） |
| 参考パイプライン | **RouteExtract**（論文） | 紙地図→GPX の4段階手法 |
| ポリラインのデコード | Google Polyline Utility / 各言語ライブラリ | `overview_polyline` を緯度経度に |

---

## 8. ASHIATO での落とし所（推奨）

1. **第一選択**: ルート検索APIの**応答座標を直接使う**（Google decode / ORS GeoJSON /
   Yahoo経路探索 / 駅すぱあと）。画像処理は不要。保存可否は各ソースの規約に従う。
2. **インタラクティブ地図**なら DOM/JS から座標を読む（§4）。
3. **画像しか無い・デジタル元が存在しない**ケースに限り §2 のパイプラインを実装。
   - まずは「自分が中心・ズームを指定して生成した静的地図」（§3の厳密逆変換）に限定すると
     精度・実装ともに堅い。任意スクショのGCP方式は精度が読みにくいので後回し。
4. 生成した GPX/GeoJSON は `trajectory.html` で重ね描き検証 → §route-matching の
   基準ルートスナップ（最近傍射影）に渡す。

---

## 9. 出典

- [RouteExtract: A Modular Pipeline for Extracting Routes from Paper Maps（arXiv 2509.11674）](https://arxiv.org/pdf/2509.11674)
- [Google Directions API — overview_polyline / 経路作成](https://blog.afi.io/blog/creating-a-route-with-the-google-maps-directions-api/)
- [Interactive Polyline Encoder Utility（Google）](https://developers.google.com/maps/documentation/utilities/polylineutility)
- [Map and Tile Coordinates（Web Mercator・slippy map、Google Maps JS API）](https://developers.google.com/maps/documentation/javascript/coordinates)
- [緯度経度↔ピクセルオフセット変換の解説](https://medium.com/@suverov.dmitriy/how-to-convert-latitude-and-longitude-coordinates-into-pixel-offsets-8461093cb9f5)
- [skeleton-tracing（LingDong）](https://github.com/LingDong-/skeleton-tracing)
- [VTracer（visioncortex, raster→vector）](https://github.com/visioncortex/vtracer)
- [OpenRouteService Geocoder（Pelias）ドキュメント](https://giscience.github.io/openrouteservice/api-reference/endpoints/geocoder/)

*関連: `routing-api-options.md`（ルート検索API比較）、`route-matching-and-import.md`
（基準ルートへのスナップ）、`battery-optimization.md` §4-2（Douglas-Peucker）。*
