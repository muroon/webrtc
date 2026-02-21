# play-from-disk

このサンプルは **ディスク上のファイル（動画/音声）をWebRTC経由でブラウザに配信する** 方法を示しています。

## 概要

ローカルに保存されたIVF（動画）やOGG（音声）ファイルを読み込み、WebRTCのメディアトラックとしてブラウザに配信します。

## 対応フォーマット

| 種類 | ファイル | コーデック |
|------|----------|------------|
| 動画 | `output.ivf` | VP8 / VP9 / AV1 |
| 音声 | `output.ogg` | Opus |

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│   Go Server     │                     │    Browser      │
│                 │                     │                 │
│  output.ivf ────┼── Video Track ─────►│  <video>        │
│  output.ogg ────┼── Audio Track ─────►│                 │
│                 │                     │                 │
│  stdin ◄────────┼── Offer (base64) ───┤                 │
│  stdout ────────┼── Answer (base64) ──►│                 │
└─────────────────┘                     └─────────────────┘
```

## 処理フロー

```
1. ファイル存在確認（output.ivf / output.ogg）
2. ブラウザからOfferを標準入力で受け取る
3. 動画/音声トラックを作成・追加
4. Answerを生成して出力
5. ICE接続完了後、ファイルを読み込みながらRTPで送信
6. ファイル終端で終了
```

## 実行方法

### 1. 入力ファイルの準備

任意の動画/音声ファイルを自前で用意します。サンプル動画を使う場合は以下のコマンドでダウンロードできます。

```bash
# サンプルディレクトリに移動
cd examples/play-from-disk

# サンプル動画をダウンロード（Big Buck Bunny）
curl -o input.mp4 https://www.w3schools.com/html/mov_bbb.mp4
```

他のサンプル動画:
```bash
# より長い動画が必要な場合
curl -L -o input.mp4 "https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4"
```

### 2. ffmpegでIVF/OGGに変換

ffmpegがインストールされていない場合は先にインストールします。

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt install ffmpeg
```

変換コマンド:

```bash
cd examples/play-from-disk

# 動画をIVFに変換（VP8/VP9/AV1）
ffmpeg -i input.mp4 -g 30 -b:v 2M output.ivf

# 音声をOGGに変換（Opus）
ffmpeg -i input.mp4 -c:a libopus -page_duration 20000 -vn output.ogg
```

| オプション | 説明 |
|------------|------|
| `-i input.mp4` | 入力ファイル |
| `-g 30` | キーフレーム間隔（30フレーム） |
| `-b:v 2M` | ビットレート（2Mbps）、必要に応じて調整可能 |
| `-c:a libopus` | Opusコーデック使用 |
| `-page_duration 20000` | OGGページ長（20ms） |
| `-vn` | 動画を除外（音声のみ） |

変換後のファイル確認:
```bash
ls -la output.*
# output.ivf と output.ogg が存在することを確認
```

### 3. サンプルを実行

```bash
# examples/play-from-disk ディレクトリで実行
go run *.go
```

プログラムが起動し、標準入力からOfferを待機する状態になります。

### 4. ブラウザでOfferを取得

1. ブラウザで [jsfiddle.net/8kup9mvn/](https://jsfiddle.net/8kup9mvn/) を開く
2. ページ内の「Copy browser SessionDescription to clipboard」ボタンをクリック
3. base64エンコードされたSDPがクリップボードにコピーされる

### 5. SDPを交換

ターミナルに戻り、コピーしたOfferを貼り付けてEnterを押します。

```bash
# 方法1: 直接貼り付け
go run *.go
# (ここでbase64文字列を貼り付けてEnter)

# 方法2: echoでパイプ
echo "eyJ0eXBlIjoib2ZmZXIiLCJzZHAiOiJ2PTANCm89LSA..." | go run *.go
```

Answerが出力されます:
```
Connection State has changed checking
Connection State has changed connected
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 6. ブラウザで再生

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリア（Browser Session Description欄の下）に貼り付け
3. 「Start Session」ボタンをクリック
4. 動画/音声が再生される

### クイックスタート（まとめ）

```bash
# 1. ディレクトリ移動
cd examples/play-from-disk

# 2. サンプル動画ダウンロード
curl -o input.mp4 https://www.w3schools.com/html/mov_bbb.mp4

# 3. 変換
ffmpeg -i input.mp4 -g 30 -b:v 2M output.ivf
ffmpeg -i input.mp4 -c:a libopus -page_duration 20000 -vn output.ogg

# 4. ブラウザで https://jsfiddle.net/8kup9mvn/ を開く
# 5. 「Copy browser SessionDescription to clipboard」をクリック
# 6. 実行（コピーしたSDPをechoでパイプ）
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 7. 出力されたAnswerをブラウザに貼り付け
# 8. 「Start Session」をクリック → 再生開始
```

## 流れまとめ

```
[自前のファイル] → [ffmpeg変換] → [Go実行] ←SDP交換→ [ブラウザ再生]
     ↓                  ↓
  input.mp4        output.ivf
                   output.ogg
```

## コード詳細

### ファイル存在確認

```go
_, err := os.Stat(videoFileName)
haveVideoFile := !os.IsNotExist(err)

_, err = os.Stat(audioFileName)
haveAudioFile := !os.IsNotExist(err)

if !haveAudioFile && !haveVideoFile {
    panic("Could not find `" + audioFileName + "` or `" + videoFileName + "`")
}
```

どちらか一方があれば動作する。

### 動画トラックの作成

```go
// IVFファイルを開いてヘッダーを読み込み
file, _ := os.Open(videoFileName)
_, header, _ := ivfreader.NewWith(file)

// FourCCからコーデックを判定
var trackCodec string
switch header.FourCC {
case "AV01":
    trackCodec = webrtc.MimeTypeAV1
case "VP90":
    trackCodec = webrtc.MimeTypeVP9
case "VP80":
    trackCodec = webrtc.MimeTypeVP8
}

// 動画トラック作成
videoTrack, _ := webrtc.NewTrackLocalStaticSample(
    webrtc.RTPCodecCapability{MimeType: trackCodec}, "video", "pion",
)
peerConnection.AddTrack(videoTrack)
```

### 動画フレームの送信

```go
// IVFファイルを読み込み
ivf, header, _ := ivfreader.NewWith(file)

// ICE接続完了を待機
<-iceConnectedCtx.Done()

// フレームレートに基づくTickerを作成
ticker := time.NewTicker(
    time.Millisecond * time.Duration(
        (float32(header.TimebaseNumerator)/float32(header.TimebaseDenominator))*1000,
    ),
)
defer ticker.Stop()

// フレームごとに送信
for ; true; <-ticker.C {
    frame, _, err := ivf.ParseNextFrame()
    if errors.Is(err, io.EOF) {
        fmt.Printf("All video frames parsed and sent")
        os.Exit(0)
    }
    videoTrack.WriteSample(media.Sample{Data: frame, Duration: time.Second})
}
```

### 音声トラックの作成と送信

```go
// 音声トラック作成
audioTrack, _ := webrtc.NewTrackLocalStaticSample(
    webrtc.RTPCodecCapability{MimeType: webrtc.MimeTypeOpus}, "audio", "pion",
)
peerConnection.AddTrack(audioTrack)

// OGGファイルを読み込み
ogg, _, _ := oggreader.NewWith(file)

// 20msごとにページを送信
ticker := time.NewTicker(oggPageDuration) // 20ms
for ; true; <-ticker.C {
    pageData, pageHeader, _ := ogg.ParseNextPage()

    // サンプル数から再生時間を計算
    sampleCount := float64(pageHeader.GranulePosition - lastGranule)
    lastGranule = pageHeader.GranulePosition
    sampleDuration := time.Duration((sampleCount/48000)*1000) * time.Millisecond

    audioTrack.WriteSample(media.Sample{Data: pageData, Duration: sampleDuration})
}
```

### RTCPの読み取り

```go
rtpSender, _ := peerConnection.AddTrack(videoTrack)

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

NACKなどのフィードバックを処理するためにRTCPパケットを読み取る必要がある。

## 重要なAPI

| API | 説明 |
|-----|------|
| `NewTrackLocalStaticSample` | サンプルベースのローカルトラック作成 |
| `AddTrack` | PeerConnectionにトラックを追加 |
| `WriteSample` | メディアサンプルを送信 |
| `ivfreader.NewWith` | IVFファイルリーダー作成 |
| `oggreader.NewWith` | OGGファイルリーダー作成 |

## time.Tickerを使う理由

```go
// NG: time.Sleepはスキュー（ずれ）が蓄積する
time.Sleep(frameDuration)

// OK: time.Tickerは正確な間隔を維持
ticker := time.NewTicker(frameDuration)
for ; true; <-ticker.C {
    // 送信処理
}
```

- `time.Sleep`はパース処理時間を補償しない
- Goのスリープには遅延問題がある（[Issue #44343](https://github.com/golang/go/issues/44343)）

## ユースケース

- ファイルベースの動画配信
- 録画コンテンツのストリーミング
- メディアサーバーの実装基盤
- VOD（Video On Demand）サービス

## 注意点

- `output.ivf` と `output.ogg` は `go run` を実行するディレクトリに配置
- 両方なくてもどちらか一方があれば動作（動画のみ、音声のみ可）
- ファイル終端まで再生すると自動終了
- ビットレート（`-b:v 2M`）はネットワーク状況に応じて調整

## 関連サンプル

- **play-from-disk-h264**: H264コーデックを使用する場合（[example-webrtc-applications](https://github.com/pion/example-webrtc-applications/tree/master/play-from-disk-h264)）
