# reflect

このサンプルは **ブラウザから受信した映像をそのままブラウザに送り返す**（ミラーリング）方法を示しています。

サーバー側処理の基盤として活用できます。

## 概要

ブラウザから送信されたWebカメラ映像をGo側で受信し、RTPパケットをそのまま送り返すことで、ブラウザに自分の映像が表示されます。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  Webcam ────────┼── Video Track ─────►│                 │
│                 │                     │  RTPパケット     │
│  <video> ◄──────┼── Video Track ◄─────│  をコピーして   │
│                 │                     │  送り返す       │
│                 │                     │                 │
│  SDP (Offer) ───┼────────────────────►│  stdin          │
│                 │◄────────────────────┼── SDP (Answer)  │
└─────────────────┘                     └─────────────────┘
```

## 処理フロー

```
1. ブラウザから映像を受信（OnTrack）
2. RTPパケットを読み取り（track.ReadRTP）
3. 出力トラックに書き込み（outputTrack.WriteRTP）
4. ブラウザで自分の映像が表示される
```

## 実行方法

### 1. サンプルディレクトリに移動

```bash
cd examples/reflect
```

### 2. ブラウザでOfferを取得

1. ブラウザで [jsfiddle.net/g643ft1k/](https://jsfiddle.net/g643ft1k/) を開く
2. カメラのアクセス許可を与える
3. 「Copy browser SDP to clipboard」をクリック

### 3. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Peer Connection State has changed: checking
Peer Connection State has changed: connected
Track has started, of type 96: video/VP8
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 4. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. 自分の映像がミラーされて表示される

## クイックスタート（まとめ）

```bash
# 1. ディレクトリ移動
cd examples/reflect

# 2. ブラウザで https://jsfiddle.net/g643ft1k/ を開く
# 3. カメラを許可
# 4. 「Copy browser SDP to clipboard」をクリック

# 5. 実行（コピーしたSDPをechoでパイプ）
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 6. 出力されたAnswerをブラウザに貼り付け
# 7. 「Start Session」をクリック → 自分の映像がミラー表示
```

## コード詳細

### MediaEngineでコーデック登録

```go
mediaEngine := &webrtc.MediaEngine{}

// VP8（動画）を登録
mediaEngine.RegisterCodec(webrtc.RTPCodecParameters{
    RTPCodecCapability: webrtc.RTPCodecCapability{
        MimeType: webrtc.MimeTypeVP8, ClockRate: 90000,
    },
    PayloadType: 96,
}, webrtc.RTPCodecTypeVideo)
```

### PLIインターセプター

```go
interceptorRegistry := &interceptor.Registry{}

// デフォルトのインターセプターを登録
webrtc.RegisterDefaultInterceptors(mediaEngine, interceptorRegistry)

// 3秒ごとにPLI（Picture Loss Indication）を送信
// → 送信側にキーフレーム生成を要求
intervalPliFactory, _ := intervalpli.NewReceiverInterceptor()
interceptorRegistry.Add(intervalPliFactory)
```

### 出力トラックの作成

```go
// ブラウザに送り返すためのトラック
outputTrack, _ := webrtc.NewTrackLocalStaticRTP(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeVP8}, "video", "pion",
)

// PeerConnectionにトラックを追加
rtpSender, _ := peerConnection.AddTrack(outputTrack)

// RTCPパケットを読み取る（NACK処理などに必要）
go func() {
    rtcpBuf := make([]byte, 1500)
    for {
        if _, _, err := rtpSender.Read(rtcpBuf); err != nil {
            return
        }
    }
}()
```

### 受信したRTPパケットを送り返す

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    fmt.Printf("Track has started, of type %d: %s \n",
        track.PayloadType(), track.Codec().MimeType)

    for {
        // RTPパケットを受信
        rtp, _, err := track.ReadRTP()
        if err != nil {
            panic(err)
        }

        // そのまま出力トラックに書き込み（送り返す）
        if err := outputTrack.WriteRTP(rtp); err != nil {
            panic(err)
        }
    }
})
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `NewTrackLocalStaticRTP` | RTPパケットベースのローカルトラック作成 |
| `AddTrack` | PeerConnectionにトラックを追加 |
| `OnTrack` | リモートトラック受信時のコールバック |
| `track.ReadRTP` | RTPパケットを読み取り |
| `outputTrack.WriteRTP` | RTPパケットを送信 |

## TrackLocalStaticRTP vs TrackLocalStaticSample

| トラックタイプ | 用途 | 入力 |
|----------------|------|------|
| `TrackLocalStaticRTP` | RTPパケットをそのまま転送 | RTPパケット |
| `TrackLocalStaticSample` | メディアサンプルを送信 | 生データ（フレーム） |

reflectでは受信したRTPパケットをそのまま転送するため、`TrackLocalStaticRTP`を使用。

## save-to-disk との違い

| 項目 | reflect | save-to-disk |
|------|---------|--------------|
| 受信後の処理 | ブラウザに送り返す | ファイルに保存 |
| 使用トラック | TrackLocalStaticRTP | なし（ファイル書き込み） |
| 出力 | リアルタイム映像 | IVF/OGGファイル |
| 双方向通信 | あり（送受信） | なし（受信のみ） |

## ユースケース

- **動作確認**: WebRTC接続のテスト
- **サーバー側処理の基盤**: 映像処理を挟んで送り返す
  - 例: フィルター適用、オブジェクト検出、字幕追加
- **SFU開発の第一歩**: 受信→送信の基本パターン
- **レイテンシ測定**: 往復遅延の確認

## 拡張例：サーバー側で映像処理を行う

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    for {
        rtp, _, _ := track.ReadRTP()

        // ここで映像処理を行う
        // 例: rtp.Payload をデコード → 処理 → エンコード
        // processedRTP := processVideo(rtp)

        outputTrack.WriteRTP(rtp)  // または processedRTP
    }
})
```

## 注意点

- このサンプルは動画（VP8）のみ対応
- 音声も送り返したい場合は、Opusコーデックの登録と別トラックの追加が必要
- RTPパケットをそのまま転送しているため、SSRCは自動的に書き換えられる

## 関連サンプル

- **save-to-disk**: 受信した映像をファイルに保存
- **play-from-disk**: ファイルから映像を送信
- **broadcast**: 1つの映像を複数のクライアントに配信
