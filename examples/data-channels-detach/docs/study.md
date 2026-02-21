# data-channels-detach と data-channels の比較

## 違いの概要

唯一の違いは **DataChannelのデータ読み書きAPI**。シグナリングや接続確立の流れは同じ。

## data-channels（コールバック方式）

```go
// イベントコールバックで送受信
dataChannel.OnMessage(func(msg webrtc.DataChannelMessage) {
    fmt.Printf("Message: %s\n", string(msg.Data))
})
dataChannel.SendText(message)
```

## data-channels-detach（io.ReadWriteCloser方式）

```go
// SettingEngineでDetachを有効化（必須）
s := webrtc.SettingEngine{}
s.DetachDataChannels()
api := webrtc.NewAPI(webrtc.WithSettingEngine(s))

// OnOpen後にDetach → io.ReadWriteCloserを取得
raw, _ := dataChannel.Detach()

// 標準的なGoのio操作で読み書き
go func() {
    buf := make([]byte, 15)
    n, _ := raw.Read(buf)    // io.Reader
}()
go func() {
    raw.Write([]byte(message)) // io.Writer
}()
```

## 比較表

| 項目 | data-channels | data-channels-detach |
|---|---|---|
| PeerConnection作成 | `webrtc.NewPeerConnection()` | `api.NewPeerConnection()`（SettingEngine経由） |
| データ受信 | `OnMessage` コールバック | `io.Reader.Read()` |
| データ送信 | `SendText()` / `Send()` | `io.Writer.Write()` |
| API スタイル | WebRTC標準（イベント駆動） | Go標準（`io.ReadWriteCloser`） |
| 設定 | 不要 | `s.DetachDataChannels()` が必要 |

## Detachの利点

- Goの `io.Reader` / `io.Writer` インターフェースに準拠するため、`bufio.Scanner` や `io.Copy` などの標準ライブラリとそのまま組み合わせられる
- goroutineでのループ読み書きパターンが自然に書ける
- 既存のストリーム処理コードとの統合が容易

## 注意点

- `OnMessage` コールバックとDetachの**併用はできない**（コード内コメントにも記載あり）
- DetachはWebRTC標準APIから外れるPion独自の拡張機能のため、`SettingEngine` での明示的な有効化が必要
