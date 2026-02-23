# rtp-to-webrtc

このサンプルは **外部のRTPストリームをWebRTC経由でブラウザに送信する** 方法を示しています。

**rtp-forwarder の逆方向** の処理を行います。

## 概要

UDPポート5004でRTPパケットを受信し、WebRTCを通じてブラウザに映像を配信します。ffmpeg、GStreamer、IPカメラなど、RTP出力が可能な任意のソースから映像を受け取れます。

## アーキテクチャ

```
┌──────────────────┐                      ┌─────────────────┐
│  RTPソース        │                      │    Go Server    │
│                  │                      │                 │
│  ffmpeg          │                      │                 │
│  GStreamer   ────┼── UDP:5004 ─────────►│  listener       │
│  IPカメラ         │  (RTPパケット)        │      │          │
│  OBS             │                      │      ▼          │
│                  │                      │  videoTrack     │
└──────────────────┘                      │      │          │
                                          └──────┼──────────┘
                                                 │
                                                 │ WebRTC
                                                 ▼
                                          ┌─────────────────┐
                                          │    Browser      │
                                          │                 │
                                          │   <video>       │
                                          │                 │
                                          └─────────────────┘
```

## rtp-forwarder との比較

| 項目 | rtp-to-webrtc | rtp-forwarder |
|------|---------------|---------------|
| 入力 | UDP/RTP (port 5004) | WebRTC (ブラウザ) |
| 出力 | WebRTC (ブラウザ) | UDP/RTP (port 4000, 4002) |
| 用途 | 外部映像をブラウザで視聴 | WebRTC映像を外部アプリで視聴 |
| 方向 | RTP → WebRTC | WebRTC → RTP |

## 実行方法

### 準備

```bash
cd examples/rtp-to-webrtc
```

### 1. ブラウザでOfferを取得

1. [jsfiddle.net/z7ms3u5r/](https://jsfiddle.net/z7ms3u5r/) を開く
2. 上部のテキストエリアにあるSDPをコピー

### 2. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Connection State has changed checking
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 3. RTPストリームを送信

**ffmpegを使用（テスト映像）:**

```bash
ffmpeg -re -f lavfi -i testsrc=size=640x480:rate=30 \
  -vcodec libvpx -cpu-used 5 -deadline 1 -g 10 \
  -error-resilient 1 -auto-alt-ref 1 \
  -f rtp 'rtp://127.0.0.1:5004?pkt_size=1200'
```

**GStreamerを使用:**

```bash
gst-launch-1.0 videotestsrc ! video/x-raw,width=640,height=480,format=I420 \
  ! vp8enc error-resilient=partitions keyframe-max-dist=10 auto-alt-ref=true cpu-used=5 deadline=1 \
  ! rtpvp8pay ! udpsink host=127.0.0.1 port=5004
```

### 4. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. ブラウザに映像が表示される

## クイックスタート（まとめ）

```bash
cd examples/rtp-to-webrtc

# 1. ブラウザで https://jsfiddle.net/z7ms3u5r/ を開く
# 2. 上部のSDPをコピー

# 3. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 4. 出力されたAnswerをブラウザに貼り付け
# 5. 「Start Session」をクリック

# 6. 別ターミナルでRTPストリームを送信
ffmpeg -re -f lavfi -i testsrc=size=640x480:rate=30 \
  -vcodec libvpx -cpu-used 5 -deadline 1 -g 10 \
  -error-resilient 1 -auto-alt-ref 1 \
  -f rtp 'rtp://127.0.0.1:5004?pkt_size=1200'

# → ブラウザに映像が表示される
```

## コード詳細

### UDPリスナーの作成

```go
// port 5004 でRTPパケットを待ち受け
listener, err := net.ListenUDP("udp", &net.UDPAddr{
    IP:   net.ParseIP("127.0.0.1"),
    Port: 5004,
})

// バッファサイズを増加（パケットロス防止）
bufferSize := 300000 // 300KB
listener.SetReadBuffer(bufferSize)
```

**バッファサイズ増加の理由:**
- デフォルトのUDPバッファサイズはOSによって異なる
- 小さいとパケットロスが発生しやすい
- 300KBに増加して安定性を向上

### videoTrackの作成

```go
videoTrack, err := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8},
    "video", "pion",
)
rtpSender, err := peerConnection.AddTrack(videoTrack)
```

### RTP → WebRTC の転送ループ

```go
inboundRTPPacket := make([]byte, 1600)  // UDP MTU
for {
    // UDPからRTPパケットを読み取り
    n, _, err := listener.ReadFrom(inboundRTPPacket)
    if err != nil {
        panic(err)
    }

    // WebRTCトラックに書き込み → ブラウザへ
    if _, err = videoTrack.Write(inboundRTPPacket[:n]); err != nil {
        if errors.Is(err, io.ErrClosedPipe) {
            return  // 接続が閉じられた
        }
        panic(err)
    }
}
```

### RTCPパケットの処理

```go
go func() {
    rtcpBuf := make([]byte, 1500)
    for {
        // RTCPパケットを読み取り（NACK処理などに必要）
        if _, _, rtcpErr := rtpSender.Read(rtcpBuf); rtcpErr != nil {
            return
        }
    }
}()
```

## 他のコーデックを使用する場合

### H.264を使用

main.goの`MimeTypeVP8`を`MimeTypeH264`に変更:

```go
videoTrack, err := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeH264},
    "video", "pion",
)
```

ffmpegコマンド:

```bash
ffmpeg -re -f lavfi -i testsrc=size=640x480:rate=30 \
  -pix_fmt yuv420p -c:v libx264 -g 10 \
  -preset ultrafast -tune zerolatency \
  -f rtp 'rtp://127.0.0.1:5004?pkt_size=1200'
```

### Opus音声を使用

main.goの`MimeTypeVP8`を`MimeTypeOpus`に変更し、トラック種別も変更:

```go
audioTrack, err := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeOpus},
    "audio", "pion",
)
```

ffmpegコマンド:

```bash
ffmpeg -f lavfi -i 'sine=frequency=1000' \
  -c:a libopus -b:a 48000 -sample_fmt s16p \
  -ssrc 1 -payload_type 111 \
  -f rtp -max_delay 0 -application lowdelay \
  'rtp://127.0.0.1:5004?pkt_size=1200'
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `net.ListenUDP()` | UDPリスナーを作成 |
| `listener.SetReadBuffer()` | 受信バッファサイズを設定 |
| `listener.ReadFrom()` | UDPパケットを読み取り |
| `videoTrack.Write()` | RTPパケットをWebRTCトラックに書き込み |
| `rtpSender.Read()` | RTCPパケットを読み取り |

## パケットロス/乱れへの対処

ネットワークが不安定な場合、[SampleBuilder](https://pkg.go.dev/github.com/pion/webrtc/v3/pkg/media/samplebuilder)を使用できます:

- RTPパケットを消費してサンプルを返す
- パケットの並べ替えと遅延処理が可能
- VP8とOpusで動作（H.264は制限あり）

## ユースケース

- **IPカメラ映像のブラウザ視聴**: RTP出力のカメラをWebRTCで配信
- **ffmpeg/GStreamer出力のブラウザ表示**: 映像処理パイプラインの結果を表示
- **OBS配信のブラウザプレビュー**: OBSのRTP出力をブラウザで確認
- **レガシーシステムとの連携**: RTPしか出力できない機器をWebRTC化
- **映像変換ゲートウェイ**: RTP → WebRTC の変換サーバー

## 注意点

- ポート5004はハードコードされている（変更する場合はmain.goを編集）
- デフォルトはVP8コーデック（H.264やOpusに変更可能）
- RTCPパケットの読み取りはNACK処理に必要
- バッファサイズはパフォーマンスに影響する

## 関連サンプル

- **rtp-forwarder**: 逆方向（WebRTC → RTP）
- **play-from-disk**: ファイルからWebRTCへ
- **broadcast**: 1対多の配信
