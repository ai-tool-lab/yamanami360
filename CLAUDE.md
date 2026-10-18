# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

「山並み360°｜淵野辺」— JR淵野辺駅から見える丹沢山系・秩父/奥秩父山系を3Dパノラマで表示する、単一HTMLの静的Webアプリ。ビルド工程・パッケージ管理・テスト・Lint は存在しない。git リポジトリでもない。

- `index.html` — アプリ全体（CSS・HTML・JS すべてインライン）
- `favicon.jpg` — アイコン原画（2048px、余白付き）。`index.html` からは参照しない
- `favicon.ico` / `icon-192.png` / `apple-touch-icon.png` — `favicon.jpg` の角丸アイコン部分を切り出し、紫〜橙のグラデーションを水色〜若葉の緑系に、背景を空色のグラデーションに色変換して生成したもの（Pillow 使用）。原画を差し替えたら再生成が必要

## 実行方法

Three.js は import map 経由で unpkg CDN (`three@0.160.0`) から ES module として読み込むため、ネット接続が必要。ローカルの静的サーバーで配信して確認する:

```bash
npx serve .
```

または `python -m http.server 8000` で `http://localhost:8000/` を開く。

## アーキテクチャ（index.html 内の `<script type="module">`）

処理は上から順に1回だけ実行され、最後に `animate()` の rAF ループに入る。

1. **座標系**: 観測点 `OBSERVER`（淵野辺駅の緯度経度・地盤標高121m＋目線1.6m）を原点とした平面近似のローカル座標。`latLonToLocalXZ()` で x=東、z=南（北が −z）。Three.js の Y が標高（m）。`curvatureDrop()` で地球曲率と大気差（`REFRACTION_K=0.13`）を補正。
2. **POIデータ `ALL_POIS`**: 山・スポットの唯一のデータソース。各要素は `id, name, kana, elev, lat, lon, type ('peak'|'spot'), range ('丹沢山系'|'秩父・奥秩父山系'), desc, routes, huts[]`。起動時に `x, z, distKm, azimuth, dir16, effectiveY, angleDeg` が追記され、後で `pin`（Three.js メッシュ）と `el`（DOMラベル）も追記される。地点追加はこの配列に1要素足すだけで、ピン・ラベル・詳細パネル・ミニマップに自動反映される。
3. **地形 `buildWideTerrain()`**: 実測DEMではなく、`type==='peak'` の各POIを中心としたガウス山（σ = max(2800, elev×2.8)）の最大値＋正弦波ノイズで合成した擬似地形。メッシュは観測点からオフセット（−42000, −15000）した 110km×100km 平面。**peak を追加・移動すると地形の形も変わる**（spot は地形に影響しない）。
4. **ラベル**: 3Dピンは Three.js、山名ラベルは `#label-layer` 上の HTML 要素。毎フレーム `Vector3.project(camera)` でスクリーン座標に変換し、近接ラベルは stem の高さを伸ばして重なりを回避している。
5. **カメラ**: 位置は観測点に固定、`azDeg/pitchDeg/fov` を `target*` へ毎フレーム 0.12 で補間。ドラッグで方位・仰角、ホイールで FOV。ヘッダーのプリセットボタンは `targetAz/targetPitch/targetFov` を直接書き換えるだけ。
6. **UI描画**: フッターのコンパス（`drawCompass` — 目印の方位はハードコード）と左下ミニマップ（`drawMinimap`）は 2D canvas で毎フレーム再描画。

## 配色の規約

- 青 `--accent-blue` (#38bdf8) = 丹沢 / `type:'peak'`、オレンジ `--accent-orange` (#fb923c) = `type:'spot'` および秩父方面のプリセット・方位（az > 270°）。
- 詳細パネルは spot 選択時に `#detail-panel.theme-orange` クラスで見出し色を切り替える。
- ミニマップでは秩父の peak のみ緑 (#34d399)。
