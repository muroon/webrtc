# save-to-disk

このサンプルは **ブラウザのWebカメラ/マイク入力をWebRTC経由で受信し、ディスクに保存する** 方法を示しています。

`play-from-disk` の逆の処理です。

## 概要

ブラウザから送信されたメディアストリーム（動画/音声）をRTPパケットとして受信し、IVF/OGGファイルとしてディスクに保存します。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  Webcam ────────┼── Video Track ─────►│  → output.ivf   │
│  Microphone ────┼── Audio Track ─────►│  → output.ogg   │
│                 │                     │                 │
│  SDP (Offer) ───┼────────────────────►│  stdin          │
│                 │◄────────────────────┼── SDP (Answer)  │
└─────────────────┘                     └─────────────────┘
```

## 保存フォーマット

| 種類 | ファイル | コーデック |
|------|----------|------------|
| 動画 | `output.ivf` | VP8 |
| 音声 | `output.ogg` | Opus |

## 処理フロー

```
1. MediaEngine でVP8/Opusコーデックを登録
2. PLIインターセプターを設定（3秒ごとにキーフレーム要求）
3. ブラウザからOfferを受け取りAnswerを返す
4. OnTrackでリモートトラックを受信
5. RTPパケットをファイルに書き込み
6. 接続終了時にファイルをクローズ
```

## 実行方法

### 1. サンプルディレクトリに移動

```bash
cd examples/save-to-disk
```

### 2. ブラウザでOfferを取得

1. ブラウザで [jsfiddle.net/2nwt1vjq/](https://jsfiddle.net/2nwt1vjq/) を開く
2. カメラ/マイクのアクセス許可を与える
3. Webカメラの映像が表示されることを確認
4. 「Copy browser SDP to clipboard」をクリック

### 3. Goプログラムを実行

```bash
# コピーしたSDPをechoでパイプ
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Connection State has changed checking
Connection State has changed connected
Ctrl+C the remote client to stop the demo
Got VP8 track, saving to disk as output.ivf
Got Opus track, saving to disk as output.opus (48 kHz, 2 channels)
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 4. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. 録画が開始される

### 5. 録画を終了

```bash
# ブラウザのタブを閉じる（推奨）
# → Go側が自動的にファイルをクローズして終了
```

終了時の出力:
```
EOF
EOF
Connection State has changed closed
Done writing media files
```

### 6. ファイル確認

録画されたファイルは `go run` を実行したディレクトリに生成されます。

```bash
ls -la output.*
# output.ivf  (動画)
# output.ogg  (音声)
```

### 7. 再生確認

```bash
# ffplayで再生（ffmpegに含まれる）
ffplay output.ivf
ffplay output.ogg

# または play-from-disk で WebRTC 経由で再生
cd ../play-from-disk
cp ../save-to-disk/output.ivf .
cp ../save-to-disk/output.ogg .
echo "{ブラウザのSDP}" | go run *.go
```

## クイックスタート（まとめ）

```bash
# 1. ディレクトリ移動
cd examples/save-to-disk

# 2. ブラウザで https://jsfiddle.net/2nwt1vjq/ を開く
# 3. カメラ/マイクを許可
# 4. 「Copy browser SDP to clipboard」をクリック

# 5. 実行（コピーしたSDPをechoでパイプ）
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 6. 出力されたAnswerをブラウザに貼り付け
# 7. 「Start Session」をクリック → 録画開始

# 8. 録画終了: ブラウザのタブを閉じる
# 9. output.ivf / output.ogg が生成される
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

// Opus（音声）を登録
mediaEngine.RegisterCodec(webrtc.RTPCodecParameters{
    RTPCodecCapability: webrtc.RTPCodecCapability{
        MimeType: webrtc.MimeTypeOpus, ClockRate: 48000,
    },
    PayloadType: 111,
}, webrtc.RTPCodecTypeAudio)
```

### PLIインターセプター

```go
interceptorRegistry := &interceptor.Registry{}

// 3秒ごとにPLI（Picture Loss Indication）を送信
// → 送信側にキーフレーム生成を要求
intervalPliFactory, _ := intervalpli.NewReceiverInterceptor()
interceptorRegistry.Add(intervalPliFactory)

// デフォルトのインターセプターも登録
webrtc.RegisterDefaultInterceptors(mediaEngine, interceptorRegistry)
```

キーフレームがあることで動画がシーク可能になる。

### Transceiverの追加

```go
// 音声と動画のトラックを受信可能にする
peerConnection.AddTransceiverFromKind(webrtc.RTPCodecTypeAudio)
peerConnection.AddTransceiverFromKind(webrtc.RTPCodecTypeVideo)
```

### ファイルライター作成

```go
// OGGファイルライター（48kHz, 2チャンネル）
oggFile, _ := oggwriter.New("output.ogg", 48000, 2)

// IVFファイルライター（VP8コーデック）
ivfFile, _ := ivfwriter.New("output.ivf", ivfwriter.WithCodec("video/VP8"))
```

### リモートトラック受信時の処理

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    codec := track.Codec()
    if strings.EqualFold(codec.MimeType, webrtc.MimeTypeOpus) {
        fmt.Println("Got Opus track, saving to disk as output.ogg")
        saveToDisk(oggFile, track)
    } else if strings.EqualFold(codec.MimeType, webrtc.MimeTypeVP8) {
        fmt.Println("Got VP8 track, saving to disk as output.ivf")
        saveToDisk(ivfFile, track)
    }
})
```

### RTPパケットをファイルに書き込み

```go
func saveToDisk(writer media.Writer, track *webrtc.TrackRemote) {
    defer writer.Close()

    for {
        rtpPacket, _, err := track.ReadRTP()
        if err != nil {
            return
        }
        writer.WriteRTP(rtpPacket)
    }
}
```

### 接続終了時の処理

```go
peerConnection.OnICEConnectionStateChange(func(state webrtc.ICEConnectionState) {
    if state == webrtc.ICEConnectionStateFailed ||
       state == webrtc.ICEConnectionStateClosed {
        oggFile.Close()
        ivfFile.Close()
        fmt.Println("Done writing media files")
        peerConnection.Close()
        os.Exit(0)
    }
})
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `oggwriter.New` | OGGファイルライター作成 |
| `ivfwriter.New` | IVFファイルライター作成 |
| `AddTransceiverFromKind` | 指定タイプのトラックを受信可能にする |
| `OnTrack` | リモートトラック受信時のコールバック |
| `track.ReadRTP` | RTPパケットを読み取り |
| `writer.WriteRTP` | RTPパケットをファイルに書き込み |

## play-from-disk との関係

```
[Browser] → save-to-disk → output.ivf/ogg → play-from-disk → [Browser]
   録画                                         再生
```

保存したファイルは `play-from-disk` でWebRTC経由で再生できます。

## 注意点

- **ファイル完成にはブラウザ終了が必要**: 正しくファイルを作成するには、リモートクライアント（ブラウザ）を閉じる必要がある
- 既存の `output.ivf` / `output.ogg` は上書きされる
- カメラ/マイクのアクセス許可が必要
- 録画されたファイルは `go run` を実行したディレクトリに生成される

## 関連サンプル

- **play-from-disk**: 保存したファイルをWebRTC経由で再生
- **save-to-disk-av1**: AV1コーデックで保存する場合
- **save-to-webm**: VP8/OpusをWebMファイルに保存（[example-webrtc-applications](https://github.com/pion/example-webrtc-applications/tree/master/save-to-webm)）
