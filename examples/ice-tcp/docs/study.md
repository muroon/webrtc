# ice-tcp

このサンプルは **ICE TCP候補のみを使用してWebRTC接続を行う** 方法を示しています。

## 概要

WebRTCは通常UDPを優先的に使用しますが、一部のネットワーク環境ではUDPがブロックされています。このサンプルでは、TCP経由でのICE接続を強制する方法を示します。

## 通常の動作 vs ICE TCP

```
通常（デフォルト）:
  ICE候補 = UDP + TCP
  → UDPが優先的に使用される

ICE TCP（このサンプル）:
  ICE候補 = TCPのみ
  → UDP候補はフィルタリングされる
```

## なぜTCPが必要か？

- 一部のネットワークではUDPがブロックされている
- 企業ファイアウォールはTCP 443/80のみ許可することが多い
- UDPが不安定な環境でのフォールバック

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│   (Offer側)     │                     │   (Answer側)    │
└────────┬────────┘                     └────────┬────────┘
         │                                       │
         │          TCP Port 8443                │
         │◄─────────────────────────────────────►│
         │         (ICE TCP接続)                 │
         │                                       │
         │          HTTP Port 8080               │
         │◄─────────────────────────────────────►│
         │         (シグナリング)                │
         │                                       │
```

## 構成

| ファイル | 役割 |
|----------|------|
| `main.go` | Go側サーバー（TCP ICE設定、Answer側） |
| `index.html` | ブラウザ側（Offer側、DataChannel受信） |

## コード詳細

### Go側 - TCP ICEの設定（main.go）

```go
func main() {
    settingEngine := webrtc.SettingEngine{}

    // TCPのみを有効化（UDPを無効化）
    settingEngine.SetNetworkTypes([]webrtc.NetworkType{
        webrtc.NetworkTypeTCP4,
        webrtc.NetworkTypeTCP6,
    })

    // TCPリスナーを作成（ポート8443）
    tcpListener, err := net.ListenTCP("tcp", &net.TCPAddr{
        IP:   net.IP{0, 0, 0, 0},
        Port: 8443,
    })
    if err != nil {
        panic(err)
    }
    fmt.Printf("Listening for ICE TCP at %s\n", tcpListener.Addr())

    // TCP Muxを作成
    // 第3引数の8は読み取りバッファサイズ
    tcpMux := webrtc.NewICETCPMux(nil, tcpListener, 8)
    settingEngine.SetICETCPMux(tcpMux)

    // APIを作成
    api = webrtc.NewAPI(webrtc.WithSettingEngine(settingEngine))

    // HTTPサーバー起動
    http.Handle("/", http.FileServer(http.Dir(".")))
    http.HandleFunc("/doSignaling", doSignaling)
    panic(http.ListenAndServe(":8080", nil))
}
```

### Go側 - シグナリング処理

```go
var api *webrtc.API

func doSignaling(res http.ResponseWriter, req *http.Request) {
    // 共有APIから新しいPeerConnectionを作成
    peerConnection, err := api.NewPeerConnection(webrtc.Configuration{})
    if err != nil {
        panic(err)
    }

    peerConnection.OnICEConnectionStateChange(func(state webrtc.ICEConnectionState) {
        fmt.Printf("ICE Connection State has changed: %s\n", state.String())
    })

    // DataChannelで3秒ごとに現在時刻を送信
    peerConnection.OnDataChannel(func(d *webrtc.DataChannel) {
        d.OnOpen(func() {
            for range time.Tick(time.Second * 3) {
                if err = d.SendText(time.Now().String()); err != nil {
                    if errors.Is(err, io.ErrClosedPipe) {
                        return
                    }
                    panic(err)
                }
            }
        })
    })

    // Offer受信 → Answer作成 → 返却
    var offer webrtc.SessionDescription
    json.NewDecoder(req.Body).Decode(&offer)
    peerConnection.SetRemoteDescription(offer)

    gatherComplete := webrtc.GatheringCompletePromise(peerConnection)
    answer, _ := peerConnection.CreateAnswer(nil)
    peerConnection.SetLocalDescription(answer)
    <-gatherComplete

    response, _ := json.Marshal(*peerConnection.LocalDescription())
    res.Write(response)
}
```

### ブラウザ側（index.html）

```javascript
let pc = new RTCPeerConnection()
let dc = pc.createDataChannel('data')

// DataChannelメッセージ受信
dc.onmessage = event => {
  console.log('Received:', event.data)
}

// ICE接続状態の変化を監視
pc.oniceconnectionstatechange = () => {
  console.log('ICE State:', pc.iceConnectionState)
}

// Offer作成・送信
pc.createOffer()
  .then(offer => {
    pc.setLocalDescription(offer)
    return fetch('/doSignaling', {
      method: 'post',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(offer)
    })
  })
  .then(res => res.json())
  .then(res => pc.setRemoteDescription(res))
```

## 重要なAPI

### SetNetworkTypes

```go
settingEngine.SetNetworkTypes([]webrtc.NetworkType{
    webrtc.NetworkTypeTCP4,
    webrtc.NetworkTypeTCP6,
})
```

使用するネットワークタイプを制限。UDPを含めないことでTCP専用となる。

利用可能なNetworkType:
- `NetworkTypeUDP4` / `NetworkTypeUDP6` - UDP
- `NetworkTypeTCP4` / `NetworkTypeTCP6` - TCP

### NewICETCPMux

```go
tcpMux := webrtc.NewICETCPMux(nil, tcpListener, 8)
```

| 引数 | 説明 |
|------|------|
| 第1引数 | ロガー（nilでデフォルト） |
| 第2引数 | TCPリスナー |
| 第3引数 | 読み取りバッファサイズ |

### SetICETCPMux

```go
settingEngine.SetICETCPMux(tcpMux)
```

SettingEngineにTCP Muxを設定。このSettingEngineから作成されたAPIを使う全てのPeerConnectionがTCP経由で接続する。

## 他のサンプルとの比較

### ice-single-port との違い

| 項目 | ice-tcp | ice-single-port |
|------|---------|-----------------|
| プロトコル | TCP | UDP |
| 目的 | UDPブロック環境対応 | ポート共有 |
| Mux | `ICETCPMux` | `ICEUDPMux` |
| API | `SetICETCPMux()` | `SetICEUDPMux()` |
| ユースケース | ファイアウォール回避 | スケーラビリティ |

### 共通点

両方とも:
- `SettingEngine` を使用してカスタム設定
- Muxを使用して単一ポートで複数接続を処理可能
- グローバルな`api`変数を共有

## UDP vs TCP の特性

| 特性 | UDP | TCP |
|------|-----|-----|
| 接続性 | ブロックされやすい | 通過しやすい |
| レイテンシ | 低い | やや高い |
| 信頼性 | 再送なし | 再送あり |
| ヘッドオブラインブロッキング | なし | あり |
| ファイアウォール | 制限されがち | 許可されやすい |

## ユースケース

- **企業ネットワーク**: UDPがブロックされている環境
- **厳格なファイアウォール**: TCP 443/80のみ許可
- **プロキシ環境**: HTTP/HTTPS経由のみ許可
- **フォールバック**: UDP接続失敗時の代替手段

## 実行方法

```bash
cd examples/ice-tcp
go run *.go
```

ブラウザで http://localhost:8080 を開く。

## UI要素

- **ICE Connection States**: 接続状態の遷移履歴
- **Inbound DataChannel Messages**: Go側から送信された現在時刻

## 注意事項

- TCPはUDPより遅延が大きくなる可能性がある
- リアルタイム性が重要な場合はUDPを優先すべき
- TCPはフォールバックオプションとして使用するのが一般的
- ポート8443はICE TCPトラフィック専用、HTTPサーバーは別ポート（8080）
