# broadcast

このサンプルは **1つの映像を複数の視聴者に配信する（SFU）** 方法を示しています。

配信者は1回だけアップロードし、サーバーがそれを複数の視聴者にリレーします。

## 概要

SFU（Selective Forwarding Unit）の基本的な実装です。配信者からの映像をサーバーで受信し、それを複数の視聴者にそのまま転送します。トランスコードは行わないため、サーバーの負荷が軽く、スケーラブルです。

## アーキテクチャ

```
                        ┌─────────────────┐
                        │   Go Server     │
                        │     (SFU)       │
┌───────────┐           │                 │           ┌───────────┐
│ Publisher │──Video───►│  localTrack     │──Video───►│ Viewer 1  │
│ (配信者)   │           │       │         │           └───────────┘
└───────────┘           │       │         │           ┌───────────┐
                        │       └─────────┼──Video───►│ Viewer 2  │
                        │                 │           └───────────┘
                        │                 │           ┌───────────┐
                        │                 │──Video───►│ Viewer N  │
                        └─────────────────┘           └───────────┘
```

## 特徴

| 項目 | 説明 |
|------|------|
| 配信者のアップロード | 1回のみ |
| 視聴者数 | 無制限（サーバーリソースの範囲内） |
| サーバーの役割 | RTPパケットをリレー（トランスコードなし） |
| 帯域効率 | 配信者の帯域を節約 |

## 処理フロー

```
1. Go側でHTTPサーバー起動（ポート8080）
2. 配信者がOffer送信 → 映像トラックを受信開始
3. 受信した映像からlocalTrackを作成
4. 視聴者がOffer送信 → localTrackを共有して配信
5. 複数の視聴者が同じ映像を受信
```

## 実行方法

### 準備

```bash
cd examples/broadcast

# Goサーバーを起動
go run *.go
```

### 1. 配信者（Publisher）の接続

**ブラウザ1:**
1. [jsfiddle.net/us4h58jx/](https://jsfiddle.net/us4h58jx/) を開く
2. 「Publish a Broadcast」をクリック
3. カメラを許可
4. 「Copy browser SDP to clipboard」をクリック

**ターミナル:**
```bash
curl localhost:8080 -d "{コピーしたbase64}"
```

出力されたAnswerをコピー。

**ブラウザ1に戻る:**
1. Answerを2番目のテキストエリアに貼り付け
2. 「Start Session」をクリック
3. 配信開始

### 2. 視聴者（Viewer）の接続

**ブラウザ2:**
1. 同じURL [jsfiddle.net/us4h58jx/](https://jsfiddle.net/us4h58jx/) を開く
2. 「Join a Broadcast」をクリック
3. 「Copy browser SDP to clipboard」をクリック

**ターミナル:**
```bash
curl localhost:8080 -d "{コピーしたbase64}"
```

出力されたAnswerをコピー。

**ブラウザ2に戻る:**
1. Answerを2番目のテキストエリアに貼り付け
2. 「Start Session」をクリック
3. 配信者の映像が表示される

### 3. 追加の視聴者

同じ手順を繰り返すことで、何人でも視聴者を追加できます。

## クイックスタート（まとめ）

```bash
cd examples/broadcast

# 1. サーバー起動
go run *.go

# 2. ブラウザで https://jsfiddle.net/us4h58jx/ を開く

# === 配信者 ===
# 3. 「Publish a Broadcast」をクリック
# 4. SDPをコピーしてcurlで送信
curl localhost:8080 -d "{配信者のSDP}"
# 5. Answerをブラウザに貼り付け → Start Session

# === 視聴者（何人でも追加可能）===
# 6. 別タブで同じURLを開く
# 7. 「Join a Broadcast」をクリック
# 8. SDPをコピーしてcurlで送信
curl localhost:8080 -d "{視聴者のSDP}"
# 9. Answerをブラウザに貼り付け → Start Session
# → 配信者の映像が表示される
```

## コード詳細

### HTTPサーバーの起動

```go
func httpSDPServer(port int) chan string {
    sdpChan := make(chan string)
    http.HandleFunc("/", func(res http.ResponseWriter, req *http.Request) {
        body, _ := io.ReadAll(req.Body)
        fmt.Fprintf(res, "done")
        sdpChan <- string(body)
    })

    go func() {
        panic(http.ListenAndServe(":"+strconv.Itoa(port), nil))
    }()

    return sdpChan
}
```

### PLIインターセプターの設定

```go
// 3秒ごとにPLI（Picture Loss Indication）を送信
// → キーフレームを要求してシーク可能にする
intervalPliFactory, _ := intervalpli.NewReceiverInterceptor()
interceptorRegistry.Add(intervalPliFactory)
```

### 配信者からの映像受信とlocalTrack作成

```go
localTrackChan := make(chan *webrtc.TrackLocalStaticRTP)

peerConnection.OnTrack(func(remoteTrack *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    // 配信者から受信した映像でlocalTrackを作成
    // このトラックを全視聴者で共有する
    localTrack, _ := webrtc.NewTrackLocalStaticRTP(
        remoteTrack.Codec().RTPCodecCapability, "video", "pion",
    )
    localTrackChan <- localTrack

    // RTPパケットをリレー
    rtpBuf := make([]byte, 1400)
    for {
        i, _, _ := remoteTrack.Read(rtpBuf)
        // localTrackに書き込む → 全視聴者に配信される
        localTrack.Write(rtpBuf[:i])
    }
})
```

### 視聴者への配信

```go
localTrack := <-localTrackChan

for {
    // 視聴者からのOffer待ち
    recvOnlyOffer := webrtc.SessionDescription{}
    decode(<-sdpChan, &recvOnlyOffer)

    // 新しいPeerConnectionを作成
    peerConnection, _ := webrtc.NewPeerConnection(peerConnectionConfig)

    // 共有のlocalTrackを追加
    rtpSender, _ := peerConnection.AddTrack(localTrack)

    // RTCPパケットを読み取る（NACKなど）
    go func() {
        rtcpBuf := make([]byte, 1500)
        for {
            if _, _, err := rtpSender.Read(rtcpBuf); err != nil {
                return
            }
        }
    }()

    // シグナリング処理
    peerConnection.SetRemoteDescription(recvOnlyOffer)
    answer, _ := peerConnection.CreateAnswer(nil)
    peerConnection.SetLocalDescription(answer)

    // Answerを出力
    fmt.Println(encode(peerConnection.LocalDescription()))
}
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `NewTrackLocalStaticRTP` | RTPパケットベースのローカルトラック作成 |
| `OnTrack` | リモートトラック受信時のコールバック |
| `remoteTrack.Read` | RTPパケットを読み取り |
| `localTrack.Write` | RTPパケットを書き込み（全視聴者に配信） |
| `AddTrack` | 視聴者のPeerConnectionにトラックを追加 |

## reflect との違い

| 項目 | broadcast | reflect |
|------|-----------|---------|
| 受信者 | 複数（N人） | 1人（送信者自身） |
| 用途 | 配信・会議 | ミラーリング |
| トラック共有 | あり（localTrack） | なし |
| PeerConnection | 複数（配信者1 + 視聴者N） | 1つ |

## SFU方式の利点

```
MCU方式（トランスコード）:
  配信者 → [デコード → ミックス → エンコード] → 視聴者
  CPU負荷: 高い
  遅延: 大きい

SFU方式（このサンプル）:
  配信者 → [RTPパケット転送] → 視聴者
  CPU負荷: 低い
  遅延: 小さい
```

## ユースケース

- ライブ配信
- ウェビナー
- 会議システム（SFU方式）
- 帯域制限のある配信者向けサービス
- 大規模イベント配信

## 動作確認のポイント

- 配信者は **1人だけ**（最初に接続）
- 視聴者は **複数人** 追加可能
- 配信者のカメラ映像が全視聴者に表示される
- 配信者は1回だけアップロード、サーバーがリレー

## 注意点

- 配信者が先に接続する必要がある
- 視聴者は配信者接続後に「Join」する
- ポート変更: `go run *.go -port 8011`
- このサンプルは動画のみ（音声を追加する場合は別トラックが必要）

## 拡張のアイデア

- 音声トラックの追加
- 複数の配信者をサポート（会議システム）
- 視聴者の切断検知
- 品質に応じたビットレート調整（Simulcast）

## 関連サンプル

- **reflect**: 1対1のミラーリング
- **save-to-disk**: 映像をファイルに保存
- **play-from-disk**: ファイルから映像を配信
