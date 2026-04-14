# 🚴 RunTube

エクササイズバイク（QB-C01S）で **走っている間だけ YouTube を見られる**ページ。  
速度が設定した閾値を超えると動画が再生され、下回ると一時停止します。

**→ [https://ryotana424.github.io/run-tube/](https://ryotana424.github.io/run-tube/)**

---

## 使い方

1. **Chrome または Edge** でページを開く（iOS Safari は非対応）
2. バイクの電源を入れる
3. 「バイク接続」ボタンをタップしてペアリング
4. 再生閾値を **30 / 40 / 50 km/h** から選択
5. ペダルを漕いで閾値を超えると動画が自動再生 🎬

2回目以降は「前回のデバイスに接続」ボタンが表示され、ワンタップで接続できます。

---

## 速度が下がったら…

閾値を下回ると動画が一時停止し、**赤青パトランプエフェクト**で警告が出ます。  
再び漕ぎ始めて閾値を超えると自動で再生が再開されます。

---

## 動作環境

| 項目 | 要件 |
|---|---|
| ブラウザ | Chrome / Edge 系（Web Bluetooth 対応） |
| 接続プロトコル | HTTPS または localhost |
| 対応バイク | QB-C01S（NEXGIM / C01 プレフィックスのデバイスも可） |
| モバイル | Android Chrome ○ / iOS Safari ✗ |

---

## 仕組み

```
エクササイズバイク
      │  Web Bluetooth (FTMS / Indoor Bike Data)
      ▼
  速度 (km/h) を取得
      │
      ├─ 速度 ≥ 閾値 ＋ 1.0  →  YouTube 再生
      └─ 速度 ＜ 閾値 − 1.0  →  YouTube 停止 ＋ パトランプ警告
```

- **Bluetooth**: FTMS サービス (0x1826) の Indoor Bike Data を優先取得、CSC Measurement (0x2A5B) をフォールバックに使用
- **YouTube**: IFrame Player API で制御。再生ボタンは非表示・操作不可
- **前回デバイス記憶**: 接続成功時にデバイス名を `localStorage` へ保存

---

## ファイル構成

```
index.html                      ← メインアプリ（単一ファイル完結）
qb_c01s_web_bluetooth_test.html ← Bluetooth 接続診断ページ
plans.md                        ← 仕様書
```
