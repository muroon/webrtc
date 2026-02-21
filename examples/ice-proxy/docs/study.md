# ice-proxy

このサンプルは **HTTPプロキシ経由でTURNサーバーに接続する** WebRTCのプロキシ機能を示しています。

## 概要

| 項目 | 内容 |
|------|------|
| TURNサーバー | `pion/turn` でローカルに起動（localhost:17342） |
| プロトコル | TURN over TCP |
| プロキシ | HTTPプロキシ（HTTP CONNECT）経由でTURN接続 |
| ICEポリシー | `ICETransportPolicyRelay`（TURN経由を強制） |

## ファイル構成

| ファイル | 役割 |
|----------|------|
| `main.go` | TURN設定定義、エントリーポイント |
| `turn.go` | TURNサーバーの起動（`pion/turn`使用） |
| `answer.go` | Answer側（プロキシ+TURN使用） |
| `offer.go` | Offer側（直接接続、ループバック許可） |
| `proxy.go` | HTTPプロキシ実装 |

## アーキテクチャ

```
┌─────────────┐                      ┌─────────────┐
│   Offerer   │                      │  Answerer   │
│  (直接接続)  │                      │(プロキシ経由)│
└──────┬──────┘                      └──────┬──────┘
       │                                    │
       │                           ┌────────┴────────┐
       │                           │   HTTP Proxy    │
       │                           └────────┬────────┘
       │                                    │
       │         ┌──────────────┐           │
       └─────────┤ TURN Server  ├───────────┘
                 │ (localhost:  │
                 │   17342)     │
                 └──────────────┘
```

## 処理フロー

1. TURNサーバーを起動（localhost:17342）
2. HTTPプロキシを起動（ランダムポート）
3. Answerer: プロキシ経由でTURN接続を設定
4. Offerer: 直接接続でOffer作成、HTTP経由でSDPを交換
5. DataChannelでラウンドトリップ時間を計測

## コード詳細

### TURN設定（main.go）

```go
const (
    turnServerAddr = "localhost:17342"
    turnServerURL  = "turn:" + turnServerAddr + "?transport=tcp"
    turnUsername   = "turn_username"
    turnPassword   = "turn_password"
)
```

### TURNサーバー起動（turn.go）

```go
func newTURNServer() *turn.Server {
    tcpListener, _ := net.Listen("tcp4", turnServerAddr)

    server, _ := turn.NewServer(turn.ServerConfig{
        AuthHandler: func(_, realm string, _ net.Addr) ([]byte, bool) {
            // 認証キーを生成して返す
            return turn.GenerateAuthKey(turnUsername, realm, turnPassword), true
        },
        ListenerConfigs: []turn.ListenerConfig{
            {
                Listener: tcpListener,
                RelayAddressGenerator: &turn.RelayAddressGeneratorNone{
                    Address: "localhost",
                },
            },
        },
    })
    return server
}
```

`pion/turn` を使用してローカルにTURNサーバーを起動。TCP接続をリッスンする。

### Answer側 - プロキシ+TURN設定（answer.go）

```go
func setupAnsweringAgent() {
    // HTTPプロキシを起動
    proxyURL := newHTTPProxy()
    // プロキシダイアラーを作成
    proxyDialer := newProxyDialer(proxyURL)

    var settingEngine webrtc.SettingEngine
    // ICEプロキシダイアラーを設定
    settingEngine.SetICEProxyDialer(proxyDialer)
    api := webrtc.NewAPI(webrtc.WithSettingEngine(settingEngine))

    peerConnection, _ := api.NewPeerConnection(webrtc.Configuration{
        ICEServers: []webrtc.ICEServer{
            {
                URLs:       []string{turnServerURL},
                Username:   turnUsername,
                Credential: turnPassword,
            },
        },
        // TURN経由を強制（プロキシを使用するため必須）
        ICETransportPolicy: webrtc.ICETransportPolicyRelay,
    })

    // DataChannelでメッセージをエコーバック
    peerConnection.OnDataChannel(func(d *webrtc.DataChannel) {
        d.OnMessage(func(msg webrtc.DataChannelMessage) {
            d.SendText(string(msg.Data))
        })
    })
}
```

重要なポイント:
- `SetICEProxyDialer()` でプロキシを設定
- `ICETransportPolicyRelay` でTURN経由を強制

### Offer側 - 直接接続（offer.go）

```go
func setupOfferingAgent() {
    var settingEngine webrtc.SettingEngine
    // ループバック候補を許可（ローカルテスト用）
    settingEngine.SetIncludeLoopbackCandidate(true)
    api := webrtc.NewAPI(webrtc.WithSettingEngine(settingEngine))

    peerConnection, _ := api.NewPeerConnection(webrtc.Configuration{})

    // DataChannelで3秒ごとに現在時刻を送信
    dc, _ := peerConnection.CreateDataChannel("data-channel", nil)
    dc.OnOpen(func() {
        for range time.Tick(3 * time.Second) {
            dc.SendText(time.Now().Format(time.RFC3339Nano))
        }
    })
    // エコーバックされた時刻からラウンドトリップ時間を計算
    dc.OnMessage(func(msg webrtc.DataChannelMessage) {
        sendTime, _ := time.Parse(time.RFC3339Nano, string(msg.Data))
        log.Printf("[Offerer] Data channel round-trip time: %s", time.Since(sendTime))
    })
}
```

### HTTPプロキシ実装（proxy.go）

```go
// プロキシダイアラー（proxy.Dialerインターフェース実装）
type proxyDialer struct {
    proxyAddr string
}

func (d *proxyDialer) Dial(network, addr string) (net.Conn, error) {
    // プロキシに接続
    conn, _ := net.Dial(network, d.proxyAddr)

    // HTTP CONNECTリクエストを送信
    req := &http.Request{
        Method: http.MethodConnect,
        URL:    &url.URL{Host: addr},
        Header: http.Header{
            "Proxy-Connection": []string{"Keep-Alive"},
        },
    }
    req.Write(conn)

    // 200 OKを確認
    resp, _ := http.ReadResponse(bufio.NewReader(conn), req)
    if resp.StatusCode != http.StatusOK {
        panic("unexpected proxy status code")
    }

    return conn, nil
}
```

```go
// プロキシサーバー側のハンドラ
func proxyHandleConn(clientConn net.Conn) {
    req, _ := http.ReadRequest(bufio.NewReader(clientConn))

    // ターゲット（TURNサーバー）に接続
    targetConn, _ := net.Dial("tcp", req.URL.Host)

    // 200 OKを返す
    clientConn.Write([]byte("HTTP/1.1 200 OK\r\n\r\n"))

    // 双方向にデータをコピー（トンネリング）
    go io.Copy(clientConn, targetConn)
    go io.Copy(targetConn, clientConn)
}
```

HTTP CONNECTメソッドを使用したトンネリングプロキシを実装。

## 重要なAPI

### SetICEProxyDialer

```go
settingEngine.SetICEProxyDialer(proxyDialer)
```

ICE接続時にプロキシを経由させるためのダイアラーを設定。`proxy.Dialer`インターフェースを実装する必要がある。

### ICETransportPolicyRelay

```go
ICETransportPolicy: webrtc.ICETransportPolicyRelay
```

ICE候補をRelayタイプ（TURN経由）のみに制限。直接接続やSTUN経由の接続を無効化する。

## ユースケース

- 企業ネットワークでHTTPプロキシ経由でのみ外部接続が許可されている環境
- ファイアウォール背後でのWebRTC接続
- TURNサーバーへのアクセスをプロキシ経由で制御したい場合

## 注意事項

- このサンプルはTURN over TCPのみをサポート
- プロキシはHTTP CONNECTメソッドを使用
- `ICETransportPolicyRelay` を設定しないとプロキシが使用されない可能性がある
