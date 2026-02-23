# insertable-streams

このサンプルは **Insertable Streams（挿入可能ストリーム）** を使用して、**送信前に映像を暗号化し、ブラウザ側で復号する**方法を示しています。

## 概要

Go側で映像フレームをXOR暗号化して送信し、ブラウザ側のJavaScriptで復号して表示します。Insertable Streamsを使うことで、WebRTCの標準暗号化（DTLS-SRTP）に加えて、アプリケーション層で追加の暗号化や加工が可能になります。

## Insertable Streamsとは？

ブラウザAPIの一つで、**エンコード済みの映像/音声フレームを加工**できる機能です。

```
通常のWebRTC:
  カメラ → エンコード → SRTP暗号化 → 送信 → SRTP復号 → デコード → 表示

Insertable Streams:
  カメラ → エンコード → [カスタム処理] → SRTP暗号化 → 送信
                              ↓
                    ここでE2E暗号化やメタデータ追加が可能
```

## アーキテクチャ

```
┌─────────────────┐                           ┌─────────────────┐
│    Go Server    │                           │    Browser      │
│                 │                           │                 │
│  output.ivf     │                           │                 │
│      │          │                           │                 │
│      ▼          │                           │                 │
│  [XOR暗号化]     │                           │  [XOR復号]       │
│  frame ^= 0xAA  │── WebRTC ────────────────►│  frame ^= 0xAA  │
│      │          │  (暗号化されたフレーム)      │      │          │
│      ▼          │                           │      ▼          │
│  videoTrack     │                           │   <video>       │
│                 │                           │                 │
└─────────────────┘                           └─────────────────┘
```

## 暗号化の仕組み

### XOR暗号の特性

```
XORの性質: A ^ B ^ B = A

例:
  元データ:     0x12
  暗号化:       0x12 ^ 0xAA = 0xB8
  復号:         0xB8 ^ 0xAA = 0x12  ← 元に戻る
```

### Go側（暗号化）

```go
const cipherKey = 0xAA

// 各フレームをXOR暗号化
for i := range frame {
    frame[i] ^= cipherKey
}
```

### JavaScript側（復号）

```javascript
// Insertable Streamsで受信したフレームを復号
for (let i = 0; i < frame.length; i++) {
    frame[i] ^= 0xAA;
}
```

## 実行方法

### 準備

```bash
cd examples/insertable-streams
```

### 1. IVFファイルを作成

```bash
# 任意の動画ファイルからVP8形式のIVFを作成
ffmpeg -i input.mp4 -g 30 output.ivf

# テスト用の映像を生成する場合
ffmpeg -f lavfi -i testsrc=duration=10:size=640x480:rate=30 -g 30 output.ivf
```

### 2. ブラウザでOfferを取得

1. [jsfiddle.net/t5xoaryc/](https://jsfiddle.net/t5xoaryc/) を開く
2. 上部のテキストエリアにあるSDPをコピー
3. 「Decrypt」チェックボックスがあることを確認

### 3. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Connection State has changed checking
Connection State has changed connected
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 4. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. 映像が表示される

### 5. 復号のON/OFFを試す

- **Decrypt ON**: 正常に映像が表示される
- **Decrypt OFF**: 映像が乱れる/表示されない

## クイックスタート（まとめ）

```bash
cd examples/insertable-streams

# 1. IVFファイルを作成
ffmpeg -f lavfi -i testsrc=duration=10:size=640x480:rate=30 -g 30 output.ivf

# 2. ブラウザで https://jsfiddle.net/t5xoaryc/ を開く
# 3. 上部のSDPをコピー

# 4. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 5. 出力されたAnswerをブラウザに貼り付け
# 6. 「Start Session」をクリック
# → 映像が表示される

# 7. 「Decrypt」チェックボックスをOFF
# → 映像が乱れる（復号されていないため）
```

## コード詳細

### IVFファイルの読み込み

```go
file, err := os.Open("output.ivf")
ivf, header, err := ivfreader.NewWith(file)
```

### フレームレートに合わせた送信

```go
// フレームレートを計算してTickerを作成
ticker := time.NewTicker(
    time.Millisecond * time.Duration(
        (float32(header.TimebaseNumerator)/float32(header.TimebaseDenominator))*1000,
    ),
)

for range ticker.C {
    frame, _, err := ivf.ParseNextFrame()

    // XOR暗号化
    for i := range frame {
        frame[i] ^= cipherKey
    }

    // 送信
    videoTrack.WriteSample(media.Sample{Data: frame, Duration: time.Second})
}
```

### 接続完了を待ってから送信開始

```go
iceConnectedCtx, iceConnectedCtxCancel := context.WithCancel(context.Background())

peerConnection.OnICEConnectionStateChange(func(state webrtc.ICEConnectionState) {
    if state == webrtc.ICEConnectionStateConnected {
        iceConnectedCtxCancel()  // 接続完了を通知
    }
})

// 送信ループ内で待機
<-iceConnectedCtx.Done()  // 接続完了まで待つ
```

## 復号チェックボックスの動作

| チェック状態 | 動作 | 映像表示 |
|-------------|------|----------|
| ON | JavaScriptでXOR復号を実行 | 正常に表示 |
| OFF | 復号処理をスキップ | 乱れる/表示されない |

## 重要なAPI

| API | 説明 |
|-----|------|
| `ivfreader.NewWith()` | IVFファイルのリーダーを作成 |
| `ivf.ParseNextFrame()` | 次のフレームを読み取り |
| `videoTrack.WriteSample()` | サンプルをトラックに書き込み |
| `context.WithCancel()` | キャンセル可能なコンテキスト |

## Insertable Streamsの用途

| 用途 | 説明 |
|------|------|
| **E2E暗号化** | サーバーを経由しても内容を見られない暗号化 |
| **メタデータ挿入** | フレームにタイムスタンプや識別子を埋め込み |
| **映像差し替え** | 完全に別の映像フィードを挿入 |
| **ウォーターマーク** | 動的な透かしの追加 |
| **コンテンツ保護** | DRMのような保護機構の実装 |

## E2E暗号化の利点

```
通常のWebRTC:
  クライアントA → [SRTP] → サーバー → [SRTP] → クライアントB
                           ↑
                      サーバーは復号可能

E2E暗号化（Insertable Streams）:
  クライアントA → [E2E暗号化] → [SRTP] → サーバー → [SRTP] → [E2E復号] → クライアントB
                                         ↑
                                    サーバーは復号不可
```

## 注意点

- XOR暗号は**デモ用の簡易暗号**（実用にはAES-GCM等を使用）
- Insertable StreamsのブラウザサポートはChrome/Edgeが中心
- `output.ivf`ファイルが実行ディレクトリに必要
- ファイル終了時にプログラムは自動終了する

## ブラウザサポート

| ブラウザ | サポート状況 |
|----------|-------------|
| Chrome | 対応 |
| Edge | 対応 |
| Firefox | 限定的 |
| Safari | 限定的 |

## 関連サンプル

- **play-from-disk**: ファイルからWebRTCへ送信
- **save-to-disk**: WebRTCからファイルへ保存
- **broadcast**: 1対多の配信
