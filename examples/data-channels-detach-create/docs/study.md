# data-channels-detach-create

このサンプルは **DataChannelのDetach機能** を使用して、低レベルな `io.ReadWriteCloser` インターフェースでデータチャネルを操作する方法を示しています。

## 主な特徴

1. **Detach機能**: WebRTCのDataChannelを「デタッチ」して、`pion/datachannel` の `io.ReadWriteCloser` インターフェースで直接読み書きできる

2. **Offerを作成する側**: このサンプルはOffer（SDP）を作成し、ペアとなる `data-channels-detach` からのAnswerを待つ

3. **HTTPサーバー**: ポート8080でHTTPサーバーを起動し、相手側からのAnswer SDPを受け取る

## 処理フロー

```
1. SettingEngineでDetach機能を有効化
2. PeerConnection作成
3. DataChannel作成（Offerを作成する側なので CreateDataChannel を呼ぶ）
4. Offer SDPを生成してbase64で出力
5. HTTP経由でAnswer SDPを受信
6. DataChannelがOpenしたらDetachして:
   - ReadLoop: io.Readerでメッセージ受信
   - WriteLoop: 5秒ごとにランダム文字列を送信
```

## コード詳細

### Detach機能の有効化

```go
// SettingEngineでDetach機能を有効化
s := webrtc.SettingEngine{}
s.DetachDataChannels()

// APIオブジェクトを作成
api := webrtc.NewAPI(webrtc.WithSettingEngine(s))
```

通常のWebRTC APIとは異なる動作のため、`SettingEngine` で明示的に有効化する必要がある。

### DataChannelの作成とDetach

```go
// DataChannelを作成（Offer側）
dataChannel, err := peerConnection.CreateDataChannel("", nil)

dataChannel.OnOpen(func() {
    // DataChannelをデタッチしてio.ReadWriteCloserを取得
    raw, dErr := dataChannel.Detach()

    // io.Readerとして読み込み
    go ReadLoop(raw)

    // io.Writerとして書き込み
    go WriteLoop(raw)
})
```

### ReadLoop - メッセージ受信

```go
func ReadLoop(d io.Reader) {
    for {
        buffer := make([]byte, messageSize)
        n, err := d.Read(buffer)
        if err != nil {
            fmt.Println("Datachannel closed; Exit the readloop:", err)
            return
        }
        fmt.Printf("Message from DataChannel: %s\n", string(buffer[:n]))
    }
}
```

標準の `io.Reader` インターフェースでデータを読み取る。

### WriteLoop - メッセージ送信

```go
func WriteLoop(d io.Writer) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    for range ticker.C {
        message, _ := randutil.GenerateCryptoRandomString(messageSize, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")
        d.Write([]byte(message))
    }
}
```

5秒ごとにランダムな文字列を送信。

### シグナリング（HTTPサーバー）

```go
func httpSDPServer(port int) chan string {
    sdpChan := make(chan string)
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        body, _ := io.ReadAll(r.Body)
        sdpChan <- string(body)
    })
    go func() {
        panic(http.ListenAndServe(":"+strconv.Itoa(port), nil))
    }()
    return sdpChan
}
```

ポート8080でHTTPサーバーを起動し、POSTされたAnswer SDPをチャネル経由で受け取る。

## 実行方法

```bash
# data-channels-detachと組み合わせて実行
go run data-channels-detach-create/*.go | go run data-channels-detach/*.go

# 出力されたAnswer SDPをcurlでPOST
curl localhost:8080/sdp -d "BASE64_SDP"
```

## data-channels-detachとの違い

| 項目 | data-channels-detach-create | data-channels-detach |
|------|----------------------------|---------------------|
| 役割 | Offer作成側 | Answer作成側 |
| DataChannel | `CreateDataChannel()` で作成 | `OnDataChannel` で受信 |
| SDP入力 | HTTP経由でAnswer受信 | 標準入力からOffer受信 |
| SDP出力 | 標準出力にOffer出力 | 標準出力にAnswer出力 |

## ポイント

- 通常の `OnMessage` コールバックではなく、Goのイディオマティックな `io.Reader/Writer` インターフェースでデータチャネルを扱える
- Detach APIと OnMessage APIを混在させることはサポートされていない
- このサンプルはOffer側として動作し、`data-channels-detach` と組み合わせて使用する
