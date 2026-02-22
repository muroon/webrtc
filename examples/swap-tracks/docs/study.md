# swap-tracks

このサンプルは **複数の入力トラックを1つの出力トラックに5秒ごとに切り替えて送信する** 方法を示しています。

## 概要

ブラウザから複数のトラックを受信し、サーバー側で5秒ごとに出力するトラックを切り替えます。会議システムのスポットライト機能やマルチカメラ切り替えの基盤となる技術です。

## Track（トラック）とは？

WebRTCにおける **Track** は、**1つのメディアストリーム（映像または音声）の単位** です。

### 基本概念

```
┌─────────────────────────────────────────┐
│            PeerConnection               │
│  ┌─────────────────────────────────┐    │
│  │     MediaStream                 │    │
│  │  ┌───────────┐  ┌───────────┐   │    │
│  │  │Video Track│  │Audio Track│   │    │
│  │  │ (カメラ)   │  │ (マイク)   │   │    │
│  │  └───────────┘  └───────────┘   │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 種類

| Track種類 | 説明 | 例 |
|-----------|------|-----|
| Video Track | 映像データ | カメラ、画面共有 |
| Audio Track | 音声データ | マイク |

### Pion WebRTCでのTrack

| 型 | 方向 | 用途 |
|----|------|------|
| `TrackRemote` | 受信 | 相手から受け取るトラック |
| `TrackLocalStaticRTP` | 送信 | RTPパケットを送信 |
| `TrackLocalStaticSample` | 送信 | メディアサンプルを送信 |

### コード例

```go
// 受信: 相手からのトラックを処理
peerConnection.OnTrack(func(track *webrtc.TrackRemote, ...) {
    // track.Kind() → Video or Audio
    // track.ReadRTP() → RTPパケットを読み取り
})

// 送信: トラックを作成して追加
videoTrack, _ := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8},
    "video",  // Track ID
    "pion",   // Stream ID
)
peerConnection.AddTrack(videoTrack)
```

### TrackとRTPの関係

```
Track = 論理的なメディアの流れ
  ↓
RTPパケット = 実際に送受信されるデータ単位
```

1つのTrackは複数のRTPパケットで構成されます。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  Track 1 ───────┼────────────────────►│                 │
│  Track 2 ───────┼────────────────────►│  5秒ごとに       │
│  Track 3 ───────┼────────────────────►│  切り替え        │
│                 │                     │      ↓          │
│  <video> ◄──────┼─────────────────────│  outputTrack    │
└─────────────────┘                     └─────────────────┘
```

## 切り替えの動作

```
時間経過:
  0-5秒:   Track 1 → outputTrack → Browser
  5-10秒:  Track 2 → outputTrack → Browser
  10-15秒: Track 3 → outputTrack → Browser
  15-20秒: Track 1 → outputTrack → Browser（ループ）
  ...
```

## 特徴

| 項目 | 説明 |
|------|------|
| 入力トラック | 複数（ブラウザから送信） |
| 出力トラック | 1つ |
| 切り替え間隔 | 5秒ごと |
| 切り替え方式 | ラウンドロビン |

## 処理フロー

```
1. 複数のトラックを受信（OnTrack）
2. 各トラックのRTPパケットをチャネルに送信（現在のトラックのみ）
3. チャネルからパケットを取り出し、出力トラックに書き込み
4. 5秒ごとに currTrack を更新して切り替え
5. 切り替え時にPLI送信（キーフレーム要求）
```

## 実行方法

### 準備

```bash
cd examples/swap-tracks
```

### 1. ブラウザでOfferを取得

1. [jsfiddle.net/1rx5on86/](https://jsfiddle.net/1rx5on86/) を開く
2. カメラを許可
3. 「Copy browser SDP to clipboard」をクリック

### 2. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Track has started, of type 96: video/VP8
Track has started, of type 96: video/VP8
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
Waiting for connection
Waiting 5 seconds then changing...
Switched to track #2
Waiting 5 seconds then changing...
Switched to track #1
...
```

### 3. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. 映像が5秒ごとに切り替わる

## クイックスタート（まとめ）

```bash
cd examples/swap-tracks

# 1. ブラウザで https://jsfiddle.net/1rx5on86/ を開く
# 2. カメラを許可
# 3. 「Copy browser SDP to clipboard」をクリック

# 4. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 5. 出力されたAnswerをブラウザに貼り付け
# 6. 「Start Session」をクリック
# → 5秒ごとに映像ソースが切り替わる
```

## コード詳細

### 出力トラックの作成

```go
// 1つの出力トラックを作成（全ての入力トラックをここに集約）
outputTrack, _ := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8},
    "video", "pion",
)

rtpSender, _ := peerConnection.AddTrack(outputTrack)

// RTCPパケットを読み取る
go func() {
    rtcpBuf := make([]byte, 1500)
    for {
        if _, _, err := rtpSender.Read(rtcpBuf); err != nil {
            return
        }
    }
}()
```

### トラック管理の変数

```go
// 現在出力中のトラック番号
currTrack := 0
// 受信したトラックの総数
trackCount := 0
// パケット用のバッファ付きチャネル
packets := make(chan *rtp.Packet, 60)
```

### トラック受信と振り分け

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    fmt.Printf("Track has started, of type %d: %s\n", track.PayloadType(), track.Codec().MimeType)

    // このトラックの番号を記録
    trackNum := trackCount
    trackCount++

    // タイムスタンプの差分計算用
    var lastTimestamp uint32
    // 現在このトラックが出力中かどうか
    var isCurrTrack bool

    for {
        rtp, _, _ := track.ReadRTP()

        // タイムスタンプを差分に変換（切り替え時の連続性のため）
        oldTimestamp := rtp.Timestamp
        if lastTimestamp == 0 {
            rtp.Timestamp = 0
        } else {
            rtp.Timestamp -= lastTimestamp
        }
        lastTimestamp = oldTimestamp

        // 現在のトラックの場合のみチャネルに送信
        if currTrack == trackNum {
            // 切り替わった直後ならPLI送信
            if !isCurrTrack {
                isCurrTrack = true
                if track.Kind() == webrtc.RTPCodecTypeVideo {
                    peerConnection.WriteRTCP([]rtcp.Packet{
                        &rtcp.PictureLossIndication{MediaSSRC: uint32(track.SSRC())},
                    })
                }
            }
            packets <- rtp
        } else {
            isCurrTrack = false
        }
    }
})
```

### パケットの出力処理

```go
go func() {
    var currTimestamp uint32
    for i := uint16(0); ; i++ {
        packet := <-packets

        // 差分タイムスタンプを累積して連続したタイムスタンプに復元
        currTimestamp += packet.Timestamp
        packet.Timestamp = currTimestamp

        // シーケンス番号を連続して付け直し
        packet.SequenceNumber = i

        // 出力トラックに書き込み
        outputTrack.WriteRTP(packet)
    }
}()
```

### 5秒ごとのトラック切り替え

```go
for {
    // トラックが1つもなければスキップ
    if trackCount == 0 {
        continue
    }

    fmt.Printf("Waiting 5 seconds then changing...\n")
    time.Sleep(5 * time.Second)

    // ラウンドロビンで次のトラックへ
    if currTrack == trackCount-1 {
        currTrack = 0  // 最後なら最初に戻る
    } else {
        currTrack++
    }
    fmt.Printf("Switched to track #%v\n", currTrack+1)
}
```

## タイムスタンプとシーケンス番号の処理

トラック切り替え時に映像が途切れないようにするための重要な処理:

```
問題:
  Track1: timestamp=1000, seq=100
  Track2: timestamp=5000, seq=200  ← 切り替え時にジャンプ
  → デコーダーが混乱

解決策:
  1. 各トラックのタイムスタンプを差分に変換
  2. 出力時に累積して連続したタイムスタンプを生成
  3. シーケンス番号は出力時に連番を付け直し
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `track.ReadRTP()` | RTPパケットを読み取り |
| `outputTrack.WriteRTP()` | RTPパケットを書き込み |
| `WriteRTCP` | PLI（キーフレーム要求）を送信 |
| `rtp.Timestamp` | RTPタイムスタンプ |
| `rtp.SequenceNumber` | RTPシーケンス番号 |

## simulcastとの違い

| 項目 | swap-tracks | simulcast |
|------|-------------|-----------|
| 入力 | 複数の異なるトラック | 1トラックの複数品質 |
| 出力 | 1トラック（切り替え） | 3トラック（全品質） |
| 用途 | ソース切り替え | 品質選択 |
| RID | 使用しない | 使用する（q/h/f） |

## ユースケース

- **スポットライト機能**: 会議でアクティブスピーカーに切り替え
- **マルチカメラ配信**: 複数カメラ間の自動切り替え
- **監視カメラ**: 複数カメラのローテーション表示
- **ピクチャインピクチャの主画面切り替え**

## 注意点

- 切り替え時にPLIを送信してキーフレームを要求する
- タイムスタンプとシーケンス番号を連続させる必要がある
- パケット用のチャネルにバッファを持たせている（60パケット）
- 切り替え間隔は5秒固定（実用時はイベント駆動に変更可能）

## 拡張のアイデア

- 音声トラックの同期切り替え
- イベント駆動の切り替え（発言検知など）
- 切り替え時のフェード効果
- 複数出力トラックへの配信

## 関連サンプル

- **simulcast**: 1トラックの複数品質ストリーム
- **broadcast**: 1対多の配信
- **reflect**: 映像のミラーリング
