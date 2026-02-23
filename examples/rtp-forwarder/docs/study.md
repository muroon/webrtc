# rtp-forwarder

このサンプルは **WebRTCで受信したメディアをUDP/RTPとして外部アプリに転送する** 方法を示しています。

VLC、ffmpeg、Twitchなどのサードパーティアプリで再生/配信できます。

## 概要

ブラウザからWebRTC経由で受信した映像/音声を、ローカルのUDPポートにRTPパケットとして転送します。これにより、WebRTC非対応のアプリケーションでもメディアを受信・処理できます。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  Webcam ────────┼── WebRTC ──────────►│                 │
│  Microphone ────┼── (SRTP暗号化)       │                 │
│                 │                     │      │          │
└─────────────────┘                     │      ▼          │
                                        │  UDP/RTP転送     │
                                        │      │          │
                                        └──────┼──────────┘
                                               │
                    ┌──────────────────────────┼──────────────┐
                    │                          │              │
                    ▼                          ▼              ▼
              ┌──────────┐              ┌──────────┐    ┌──────────┐
              │   VLC    │              │  ffmpeg  │    │  Twitch  │
              │ Port 4000│              │ ffplay   │    │  RTMP    │
              │ Port 4002│              │          │    │          │
              └──────────┘              └──────────┘    └──────────┘
```

## 転送ポート

| メディア | UDPポート | PayloadType | コーデック |
|----------|-----------|-------------|-----------|
| 音声 | 4000 | 111 | Opus |
| 映像 | 4002 | 96 | VP8 |

## 実行方法

### 準備

```bash
cd examples/rtp-forwarder
```

### 1. ブラウザでOfferを取得

1. [jsfiddle.net/fm7btvr3/](https://jsfiddle.net/fm7btvr3/) を開く
2. カメラ/マイクを許可
3. 「Copy browser SDP to clipboard」をクリック

### 2. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Connection State has changed checking
Connection State has changed connected
Ctrl+C the remote client to stop the demo
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 3. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. RTP転送が開始される

### 4. 外部アプリで再生

```bash
# VLC
vlc rtp-forwarder.sdp

# ffplay（低遅延オプション付き）
ffplay -i rtp-forwarder.sdp -protocol_whitelist file,udp,rtp -fflags nobuffer -flags low_delay -framedrop

# ffprobe（ストリーム情報確認）
ffprobe -i rtp-forwarder.sdp -protocol_whitelist file,udp,rtp
```

### 5. Twitch配信（オプション）

```bash
ffmpeg -protocol_whitelist file,udp,rtp -i rtp-forwarder.sdp \
  -c:v libx264 -preset veryfast -b:v 3000k -maxrate 3000k -bufsize 6000k \
  -pix_fmt yuv420p -g 50 \
  -c:a aac -b:a 160k -ac 2 -ar 44100 \
  -f flv rtmp://live.twitch.tv/app/$STREAM_KEY
```

## クイックスタート（まとめ）

```bash
cd examples/rtp-forwarder

# 1. ブラウザで https://jsfiddle.net/fm7btvr3/ を開く
# 2. カメラ/マイクを許可
# 3. 「Copy browser SDP to clipboard」をクリック

# 4. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 5. 出力されたAnswerをブラウザに貼り付け
# 6. 「Start Session」をクリック

# 7. VLCまたはffplayで再生
vlc rtp-forwarder.sdp
# または
ffplay -i rtp-forwarder.sdp -protocol_whitelist file,udp,rtp
```

## コード詳細

### コーデック登録

```go
mediaEngine := &webrtc.MediaEngine{}

// VP8（映像）を登録
mediaEngine.RegisterCodec(webrtc.RTPCodecParameters{
    RTPCodecCapability: webrtc.RTPCodecCapability{
        MimeType: webrtc.MimeTypeVP8, ClockRate: 90000,
    },
}, webrtc.RTPCodecTypeVideo)

// Opus（音声）を登録
mediaEngine.RegisterCodec(webrtc.RTPCodecParameters{
    RTPCodecCapability: webrtc.RTPCodecCapability{
        MimeType: webrtc.MimeTypeOpus, ClockRate: 48000,
    },
}, webrtc.RTPCodecTypeAudio)
```

#### コーデック（Codec）とは？

**Codec = Coder + Decoder**（符号化 + 復号化）の略で、メディアデータを圧縮・展開するアルゴリズムです。

**なぜ必要？**

```
生データ（非圧縮）:
  1秒の映像 = 約 150MB（1080p, 30fps）
  ↓
  ネットワーク転送に現実的ではない

コーデックで圧縮:
  1秒の映像 = 約 0.5MB（VP8, 4Mbps）
  ↓
  ネットワーク転送可能
```

**WebRTCでよく使うコーデック**

| 種類 | コーデック | 特徴 |
|------|-----------|------|
| 映像 | VP8 | 標準的、広くサポート |
| 映像 | VP9 | VP8より高圧縮 |
| 映像 | H.264 | ハードウェアサポート豊富 |
| 映像 | AV1 | 最新、最高圧縮率 |
| 音声 | Opus | 標準、低遅延 |
| 音声 | G.711 | 電話品質、シンプル |

**処理の流れ**

```
送信側:
  カメラ映像 → [エンコード(VP8)] → RTPパケット → ネットワーク

受信側:
  ネットワーク → RTPパケット → [デコード(VP8)] → 画面表示
```

**コード登録の意味**

```go
// 「このPeerConnectionはVP8とOpusを使います」と宣言
mediaEngine.RegisterCodec(..., webrtc.MimeTypeVP8, ...)
mediaEngine.RegisterCodec(..., webrtc.MimeTypeOpus, ...)
```

登録しないコーデックは使用できません。相手と共通のコーデックがないと通信できません。

**PayloadTypeとは？**

```go
{port: 4000, payloadType: 111}  // Opus
{port: 4002, payloadType: 96}   // VP8
```

RTPパケット内でコーデックを識別するための番号です。

### コーデック登録後の処理フロー

ffplayでストリーミングが再生されるまでの全体像:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          処理の流れ                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. コーデック登録 (main.go:41-54)                                       │
│         ↓                                                               │
│  2. PLIインターセプター設定 (main.go:60-70)                              │
│         ↓                                                               │
│  3. PeerConnection作成 (main.go:77-98)                                  │
│         ↓                                                               │
│  4. Transceiver追加 (main.go:101-105) ← 音声/映像を受信可能にする        │
│         ↓                                                               │
│  5. UDP接続の準備 (main.go:107-136) ← ffplayへの出力先                   │
│         ↓                                                               │
│  6. OnTrackハンドラ設定 (main.go:141-183) ← RTP→UDP転送ループ            │
│         ↓                                                               │
│  7. SDP交換 (main.go:217-245)                                           │
│         ↓                                                               │
│  8. WebRTC接続確立 → RTPパケット受信開始                                 │
│         ↓                                                               │
│  9. ffplayがUDPポートからRTPを受信 → 再生                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**各ステップの役割:**

| ステップ | 処理 | 役割 |
|----------|------|------|
| PLIインターセプター | 3秒ごとにキーフレーム要求 | ffplayが途中から再生開始できる |
| Transceiver追加 | 音声/映像の受信を有効化 | OnTrackが呼ばれるようになる |
| UDP接続準備 | port 4000/4002への接続 | ffplayへの出力先を確保 |
| OnTrackハンドラ | RTP読み取り→UDP送信ループ | WebRTCとffplayの橋渡し |
| PayloadType変換 | ブラウザ値→SDP値に変換 | ffplayがコーデックを認識できる |

**ffplayが再生できる条件:**

| 条件 | コード上の対応 |
|------|---------------|
| UDPポートでRTPを受信 | `net.DialUDP` → port 4000, 4002 |
| PayloadTypeがSDPと一致 | `rtpPacket.PayloadType = conn.payloadType` |
| キーフレームが存在 | PLIインターセプターで定期要求 |
| コーデックがSDPと一致 | `rtp-forwarder.sdp` に VP8/Opus を記載 |

**データフロー:**

```
Browser ──[SRTP]──► Go Server ──[RTP]──► UDP:4000/4002 ──► ffplay
                         │
                         ├─ SRTP復号化（自動）
                         ├─ PayloadType変換
                         └─ UDPで転送
```

### UDP接続の準備

```go
type udpConn struct {
    conn        *net.UDPConn
    port        int
    payloadType uint8
}

// 音声と映像用のUDP接続を作成
udpConns := map[string]*udpConn{
    "audio": {port: 4000, payloadType: 111},
    "video": {port: 4002, payloadType: 96},
}

for _, conn := range udpConns {
    raddr, _ := net.ResolveUDPAddr("udp", fmt.Sprintf("127.0.0.1:%d", conn.port))
    conn.conn, _ = net.DialUDP("udp", laddr, raddr)
}
```

### WebRTCトラックをUDPに転送

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    // トラック種別（audio/video）に対応するUDP接続を取得
    conn, ok := udpConns[track.Kind().String()]
    if !ok {
        return
    }

    buf := make([]byte, 1500)
    rtpPacket := &rtp.Packet{}

    for {
        // WebRTCからRTPパケットを読み取り
        n, _, _ := track.Read(buf)

        // PayloadTypeを更新（SDPファイルと一致させる）
        rtpPacket.Unmarshal(buf[:n])
        rtpPacket.PayloadType = conn.payloadType
        n, _ = rtpPacket.MarshalTo(buf)

        // UDPで送信
        conn.conn.Write(buf[:n])
    }
})
```

### PLIインターセプター

```go
// 3秒ごとにPLI（キーフレーム要求）を送信
intervalPliFactory, _ := intervalpli.NewReceiverInterceptor()
interceptorRegistry.Add(intervalPliFactory)
```

## SDPファイル（rtp-forwarder.sdp）

```
v=0
o=- 0 0 IN IP4 127.0.0.1
s=Pion WebRTC
c=IN IP4 127.0.0.1
t=0 0
m=audio 4000 RTP/AVP 111
a=rtpmap:111 OPUS/48000/2
m=video 4002 RTP/AVP 96
a=rtpmap:96 VP8/90000
```

このファイルはVLCやffmpegがRTPストリームを受信するために必要です。

### SDPファイルの役割と実際のデータの流れ

**重要**: SDPファイルには映像/音声データは含まれていません。「どこで何を受信するか」の**設定情報**のみです。

```
┌─────────────────────────────────────────────────────────────────────┐
│  rtp-forwarder.sdp = 「指示書」                                      │
│                                                                     │
│  「ポート4000でOpus音声を、ポート4002でVP8映像を受信せよ」            │
│  （映像/音声データは含まれていない）                                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  実際のデータの流れ                                                  │
│                                                                     │
│  Go Server ──[UDP]──► port 4000 (音声) ──► ffplay                   │
│             ──[UDP]──► port 4002 (映像) ──► ffplay                   │
│                                                                     │
│  ffplayが表示するのは「UDPで受け取ったストリーム」                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 役割 | SDPファイル | UDPストリーム |
|------|-------------|---------------|
| 内容 | 設定情報（テキスト） | 映像/音声データ（バイナリ） |
| サイズ | 数百バイト | 継続的に流れる |
| 読み込み | 起動時に1回 | リアルタイムで継続 |

### SDPファイルはGoコードで生成されない

`rtp-forwarder.sdp`は**手動で作成された静的ファイル**です。main.go内のコメント:

```go
// Also update incoming packets with expected PayloadType, the browser may use
// a different value. We have to modify so our stream matches what rtp-forwarder.sdp expects
```

これは「rtp-forwarder.sdpが期待する値に合わせる」という**説明文**であり、ファイルを生成するコードではありません。

### ファイル名は自由に変更可能

```bash
# 任意のファイル名でOK
cp rtp-forwarder.sdp my-stream.sdp
ffplay -i my-stream.sdp -protocol_whitelist file,udp,rtp
```

### SDPファイルなしでも再生可能

```bash
# SDPファイルを使う方法
ffplay -i rtp-forwarder.sdp -protocol_whitelist file,udp,rtp

# SDPファイルなしで直接UDPを指定する方法（同じ結果）
ffplay -protocol_whitelist rtp,udp -i "rtp://127.0.0.1:4002"
```

SDPファイルは便利な設定ファイルですが、必須ではありません。

## 重要なAPI

| API | 説明 |
|-----|------|
| `track.Read()` | RTPパケットをバイト列として読み取り |
| `rtp.Packet.Unmarshal()` | バイト列をRTPパケットに変換 |
| `rtp.Packet.MarshalTo()` | RTPパケットをバイト列に変換 |
| `net.DialUDP()` | UDP接続を作成 |
| `conn.Write()` | UDPでデータを送信 |

## save-to-diskとの違い

| 項目 | rtp-forwarder | save-to-disk |
|------|---------------|--------------|
| 出力先 | UDP/RTP（リアルタイム） | ファイル（IVF/OGG） |
| 用途 | 外部アプリ連携 | 録画保存 |
| 再生 | VLC/ffplayでリアルタイム | 後で再生 |
| データ形式 | 生RTPパケット | コンテナ形式 |

## ユースケース

- **VLCで再生**: WebRTC映像をデスクトップで視聴
- **ffmpegで変換**: 他フォーマットへのトランスコード
- **Twitch/YouTube配信**: WebRTCからRTMPへ変換して配信
- **録画**: ffmpegでファイルに保存
- **映像処理パイプライン**: 他ツールとの連携

## 注意点

- メディアはライブ/ステートレス：コマンドの切り替えは再起動不要
- 先にAnswerをブラウザに設定してから外部アプリを起動する
- `protocol_whitelist`オプションはffmpeg/ffplayで必須
- PayloadTypeはSDPファイルと一致させる必要がある

## 低遅延再生オプション

```bash
ffplay -i rtp-forwarder.sdp -protocol_whitelist file,udp,rtp \
  -fflags nobuffer \
  -flags low_delay \
  -framedrop
```

| オプション | 効果 |
|------------|------|
| `-fflags nobuffer` | バッファリングを無効化 |
| `-flags low_delay` | 低遅延モード |
| `-framedrop` | フレームドロップ許可（遅延軽減） |

※ジッターの多いネットワークでは再生品質が低下する可能性あり

## 関連サンプル

- **save-to-disk**: ファイルに保存
- **play-from-disk**: ファイルから再生
- **broadcast**: 1対多の配信
