# simulcast

このサンプルは **Simulcast（複数品質レベルの映像同時送信）** の処理方法を示しています。

ブラウザから3つの異なる解像度の映像ストリームを受信し、それぞれをブラウザに送り返します。

## Simulcastとは？

Simulcastは、1つのカメラ映像から複数の品質レベル（解像度・ビットレート）のストリームを同時に送信する技術です。サーバー（SFU）側で視聴者の帯域に応じて適切な品質を選択して配信できます。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  Webcam ────────┼── q (低品質) ───────►│ → outputTrack_q │──►│
│    │            │                     │                 │   │
│    ├────────────┼── h (中品質) ───────►│ → outputTrack_h │──►│
│    │            │                     │                 │   │
│    └────────────┼── f (高品質) ───────►│ → outputTrack_f │──►│
│                 │                     │                 │   │
│  ┌────────────┐ │                     │                 │   │
│  │ <video> q  │◄┼─────────────────────┼─────────────────┼───┘
│  │ <video> h  │◄┼─────────────────────┼─────────────────┼───┘
│  │ <video> f  │◄┼─────────────────────┼─────────────────┼───┘
│  └────────────┘ │                     │                 │
└─────────────────┘                     └─────────────────┘
```

## 品質レベル（RID）

| RID | 品質 | 解像度（例） | ビットレート（例） |
|-----|------|------------|------------------|
| `q` | Quarter（低） | 320x180 | 150kbps |
| `h` | Half（中） | 640x360 | 500kbps |
| `f` | Full（高） | 1280x720 | 2000kbps |

※ 実際の解像度・ビットレートはブラウザの帯域推定に依存します。

## 処理フロー

```
1. 3つの出力トラック（q, h, f）を作成
2. ブラウザからSimulcastストリームを受信
3. 各ストリームをRID（q/h/f）で識別
4. 対応する出力トラックに転送
5. ブラウザで3つの映像を表示
```

## 実行方法

### 準備

```bash
cd examples/simulcast
```

### 1. ブラウザでOfferを取得

1. [jsfiddle.net/tz4d5bhj/](https://jsfiddle.net/tz4d5bhj/) を開く
2. カメラを許可
3. 「Copy browser SDP to clipboard」をクリック

### 2. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Track has started
Sending pli for stream with rid: "q", ssrc: 1234567890
Track has started
Sending pli for stream with rid: "h", ssrc: 1234567891
Track has started
Sending pli for stream with rid: "f", ssrc: 1234567892
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 3. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. 3つの品質の映像が表示される

## クイックスタート（まとめ）

```bash
cd examples/simulcast

# 1. ブラウザで https://jsfiddle.net/tz4d5bhj/ を開く
# 2. カメラを許可
# 3. 「Copy browser SDP to clipboard」をクリック

# 4. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 5. 出力されたAnswerをブラウザに貼り付け
# 6. 「Start Session」をクリック
# → 3つの品質レベルの映像が表示される
```

## コード詳細

### 3つの出力トラックを作成

```go
outputTracks := map[string]*webrtc.TrackLocalStaticRTP{}

// 低品質（q = Quarter）
outputTrack, _ := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8},
    "video_q", "pion_q",
)
outputTracks["q"] = outputTrack

// 中品質（h = Half）
outputTrack, _ = webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8},
    "video_h", "pion_h",
)
outputTracks["h"] = outputTrack

// 高品質（f = Full）
outputTrack, _ = webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8},
    "video_f", "pion_f",
)
outputTracks["f"] = outputTrack
```

### Transceiverの設定

```go
// 受信用（recvonly）
peerConnection.AddTransceiverFromKind(
    webrtc.RTPCodecTypeVideo,
    webrtc.RTPTransceiverInit{Direction: webrtc.RTPTransceiverDirectionRecvonly},
)

// 送信用（sendonly）- 各品質レベル
peerConnection.AddTransceiverFromTrack(
    outputTracks["q"],
    webrtc.RTPTransceiverInit{Direction: webrtc.RTPTransceiverDirectionSendonly},
)
peerConnection.AddTransceiverFromTrack(
    outputTracks["h"],
    webrtc.RTPTransceiverInit{Direction: webrtc.RTPTransceiverDirectionSendonly},
)
peerConnection.AddTransceiverFromTrack(
    outputTracks["f"],
    webrtc.RTPTransceiverInit{Direction: webrtc.RTPTransceiverDirectionSendonly},
)
```

### 受信したストリームをRIDで振り分け

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    fmt.Println("Track has started")

    // RID（q/h/f）を取得
    rid := track.RID()

    // 3秒ごとにPLI（キーフレーム要求）を送信
    if track.Kind() == webrtc.RTPCodecTypeVideo {
        go func() {
            ticker := time.NewTicker(3 * time.Second)
            defer ticker.Stop()
            for range ticker.C {
                fmt.Printf("Sending pli for stream with rid: %q, ssrc: %d\n", track.RID(), track.SSRC())
                peerConnection.WriteRTCP([]rtcp.Packet{
                    &rtcp.PictureLossIndication{MediaSSRC: uint32(track.SSRC())},
                })
            }
        }()
    }

    // RTPパケットを対応する出力トラックに転送
    for {
        packet, _, _ := track.ReadRTP()
        outputTracks[rid].WriteRTP(packet)
    }
})
```

### RTCPパケットの処理

```go
processRTCP := func(rtpSender *webrtc.RTPSender) {
    rtcpBuf := make([]byte, 1500)
    for {
        if _, _, err := rtpSender.Read(rtcpBuf); err != nil {
            return
        }
    }
}

for _, rtpSender := range peerConnection.GetSenders() {
    go processRTCP(rtpSender)
}
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `track.RID()` | ストリームのRID（q/h/f）を取得 |
| `track.SSRC()` | ストリームのSSRC（同期ソース識別子）を取得 |
| `AddTransceiverFromTrack` | 方向を指定してトラックを追加 |
| `WriteRTCP` | RTCPパケット（PLIなど）を送信 |
| `RTPTransceiverDirectionRecvonly` | 受信のみ |
| `RTPTransceiverDirectionSendonly` | 送信のみ |

## broadcastとの違い

| 項目 | simulcast | broadcast |
|------|-----------|-----------|
| 映像ストリーム数 | 3（q/h/f） | 1 |
| 用途 | 品質選択可能な配信 | 単一品質の配信 |
| 複雑度 | 高い | 低い |
| 帯域適応 | サーバー側で選択可能 | なし |

## Simulcastの利点

```
通常の配信:
  配信者 → [単一品質] → 全視聴者（帯域が足りない視聴者は視聴困難）

Simulcast:
  配信者 → [高品質] → 帯域に余裕のある視聴者
         → [中品質] → 普通の視聴者
         → [低品質] → 帯域が限られた視聴者
```

## ユースケース

- **適応的ビットレート配信**: 視聴者の帯域に応じて品質を切り替え
- **会議システム**: スピーカーは高品質、その他は低品質
- **帯域の効率的利用**: サーバーで品質を選択して配信
- **モバイル対応**: 回線状況に応じて品質を動的に変更

## 注意点

- ブラウザは帯域が十分な場合のみ高品質ストリームを送信する
- 帯域推定は `chrome://webrtc-internals` の `VideoBwe` で確認可能
- 3つのストリーム全てが常に送信されるわけではない
- PLIを定期的に送信してキーフレームを要求する必要がある

## 関連サンプル

- **broadcast**: 単一品質の配信
- **reflect**: 映像のミラーリング
- **rtp-forwarder**: RTPパケットの転送
