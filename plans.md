# RunTube — 仕様書

> エクササイズバイク（QB-C01S）で走っている間だけ YouTube を見られるページ

---

## 概要

Web Bluetooth で QB-C01S からリアルタイムに速度を取得し、設定した閾値を超えている間だけ YouTube 動画を再生できる単一 HTML ファイル。  
速度が閾値を下回ると動画を一時停止し、閾値を超えると自動再生を再開する。  
GitHub Pages で配信するため、サーバーサイド処理は一切持たない。

---

## ファイル構成

```
run-tube/
├── index.html          ← メイン成果物（単一ファイル完結）
├── plans.md            ← 本仕様書
└── qb_c01s_web_bluetooth_test.html  ← 参考：接続テストページ
```

---

## 機能要件

### 1. Bluetooth 接続

| 項目 | 内容 |
|---|---|
| 対象デバイス | QB-C01S（NEXGIM / C01 プレフィックスも許容） |
| 接続方法 | 名前フィルタ接続 / 全デバイスから選ぶ（両方のボタンを用意） |
| 使用サービス | FTMS (0x1826) の Indoor Bike Data (0x2AD2) を第一候補 |
| フォールバック | CSC Measurement (0x2A5B) でケイデンスから疑似速度を算出 |
| 切断検知 | `gattserverdisconnected` イベントで動画を即停止 |

### 2. 速度取得

- **FTMS / Indoor Bike Data** から `Instantaneous Speed` フィールド (uint16, 0.01 km/h 分解能) を読む
- FTMS が使えない場合は **CSC Measurement** のクランク回転数差分からケイデンス (rpm) を算出し、  
  設定ホイール周長を掛けて疑似速度 (km/h) を算出する
- 速度データは 1 秒ごと（Notify の受信頻度に依存）に更新

### 3. YouTube 埋め込み

- YouTube IFrame Player API (`https://www.youtube.com/iframe_api`) を使用
- **動画はプレイリストに固定**（URL 入力欄は設けない）
  - 起動動画ID: `QWDayFgPDjQ`
  - プレイリストID: `RDEMI0V0e34vA6znf4KLMCIbbQ`
- `playerVars`:
  - `autoplay: 0`（初期は停止）
  - `controls: 1`（ユーザーが手動操作できる）
  - `listType: 'playlist'`
  - `list: 'RDEMI0V0e34vA6znf4KLMCIbbQ'`
  - `rel: 0`

### 4. 速度による再生制御

```
速度 >= 閾値  →  player.playVideo()
速度 < 閾値   →  player.pauseVideo()
```

- 閾値はスライダー＋数値入力で 0〜40 km/h の範囲で設定（デフォルト 10 km/h）
- 判定は速度データ受信ごとに実施
- Bluetooth が未接続の間は制御しない（手動操作を許可）

### 5. UI 状態

| 状態 | 表示 |
|---|---|
| Bluetooth 未接続 | オーバーレイ「Bluetoothで接続してください」を動画上に表示、動画操作は無効化 |
| 接続済み・速度 >= 閾値 | 通常再生中、オーバーレイ非表示 |
| 接続済み・速度 < 閾値 | 一時停止、オーバーレイ「ペダルを漕いでください 🚴」を表示 |
| 切断 | オーバーレイ「接続が切れました」を表示、動画一時停止 |

---

## 非機能要件

- **単一 HTML ファイル**: CSS・JS をすべてインライン化、外部アセット不要
- **GitHub Pages 対応**: HTTPS 環境で動作（Web Bluetooth は HTTPS 必須）
- **レスポンシブ**: スマートフォン横持ち・PC 両対応（YouTube プレーヤーは 16:9 維持）
- **ブラウザ**: Chrome / Edge 系（Web Bluetooth 対応ブラウザ）

---

## 画面レイアウト

```
┌─────────────────────────────────────────────┐
│  🚴 RunTube                                  │
├─────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────┐ │
│  │                                         │ │
│  │        YouTube プレーヤー (16:9)         │ │
│  │   [オーバーレイ: 状態メッセージ]          │ │
│  │                                         │ │
│  └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│  速度: 12.5 km/h  ケイデンス: 75 rpm         │
│  閾値: [──●────] 10 km/h  状態: ✅ 走行中   │
├─────────────────────────────────────────────┤
│  [Bluetooth 接続]  [全デバイス]  [切断]       │
└─────────────────────────────────────────────┘
```

---

## 技術スタック

- **言語**: HTML / CSS / Vanilla JS（フレームワーク不使用）
- **API**:
  - [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API)
  - [YouTube IFrame Player API](https://developers.google.com/youtube/iframe_api_reference)
- **ホスティング**: GitHub Pages（静的配信）

---

## Bluetooth 実装方針（`qb_c01s_web_bluetooth_test.html` からの移植）

既存テストページで動作確認済みの以下のロジックをそのまま採用する。

| 移植元 | 移植内容 |
|---|---|
| `connect()` | Bluetooth 接続処理 |
| `inspectServices()` | 省略（本番では不要） |
| `subscribeKnownCharacteristics()` | FTMS / CSC への通知登録 |
| `parseIndoorBikeData()` | 速度・ケイデンス・パワー解析 |
| `parseCscMeasurement()` | CSC からケイデンス算出 |
| `onDisconnected()` | 切断時処理 |

---

## 実装ステップ

1. `index.html` を新規作成
2. Bluetooth 接続・速度取得ロジックを移植
3. YouTube IFrame Player API を組み込み
4. 速度閾値判定 → `playVideo` / `pauseVideo` 制御を実装
5. オーバーレイ UI を実装
6. レスポンシブ対応 CSS を整備
7. GitHub Pages へデプロイ（`gh-pages` ブランチ or `main` の `/docs` フォルダ）

---

## 未決事項・検討事項

| 項目 | 内容 |
|---|---|
| 閾値ヒステリシス | 速度がちょうど閾値付近で細かく停止/再生を繰り返す場合、`±0.5 km/h` のヒステリシスを設けることを検討 |
| ページ離脱時の自動切断 | `beforeunload` でBluetoothを切断するか否か |
| モバイル対応 | iOS Safari は Web Bluetooth 非対応のため、Android Chrome に限定する旨を明記 |
| 動画の音量 | 走行中は音量を上げ、停止時に下げるオプション（音量フェードイン/アウト）を将来機能として検討 |
