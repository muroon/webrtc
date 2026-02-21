# ice-single-port

このサンプルは **複数のPeerConnectionを単一ポートで処理する** 方法を示しています。

## 概要

Pion WebRTCはデフォルトでグローバルな状態を持たないため、各PeerConnectionは独自のポートを使用します。`SettingEngine`を使用することで、複数のPeerConnection間で状態を共有し、単一ポートで全ての接続を処理できます。

## 通常の動作 vs Single Port

```
通常（デフォルト）:
  PeerConnection 1 → UDP Port 50001
  PeerConnection 2 → UDP Port 50002
  PeerConnection 3 → UDP Port 50003
  ...（接続数分のポートが必要）

Single Port（このサンプル）:
  PeerConnection 1 ─┐
  PeerConnection 2 ─┼→ UDP Port 8443（共有）
  PeerConnection 3 ─┘
  ...（全て同じポート）
```

## アーキテクチャ

```
┌──────────────────────────────────────────────────────┐
│                    Go Server                          │
│  ┌─────────────────────────────────────────────────┐ │
│  │              SettingEngine                       │ │
│  │  ┌─────────────────────────────────────────┐    │ │
│  │  │     ICE UDP Mux (Port 8443)             │    │ │
│  │  │  ┌───────────┬───────────┬───────────┐  │    │ │
│  │  │  │   PC 1    │   PC 2    │   PC 3    │  │    │ │
│  │  │  └───────────┴───────────┴───────────┘  │    │ │
│  │  └─────────────────────────────────────────┘    │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
                         ▲
                         │ UDP Port 8443
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
┌───┴───┐          ┌─────┴────┐         ┌─────┴────┐
│Browser│          │ Browser  │         │ Browser  │
│  PC1  │          │   PC2    │         │   PC3    │
└───────┘          └──────────┘         └──────────┘
```

## 構成

| ファイル | 役割 |
|----------|------|
| `main.go` | Go側サーバー（UDPMux設定、複数PeerConnection管理） |
| `index.html` | ブラウザ側（10個のPeerConnectionを作成） |

## コード詳細

### Go側 - UDPMuxの設定（main.go）

```go
var api *webrtc.API

func main() {
    // SettingEngineを作成
    settingEngine := webrtc.SettingEngine{}

    // ポート8443でUDPMuxを作成
    // 全WebRTCトラフィックをこのポートでリッスン
    mux, err := ice.NewMultiUDPMuxFromPort(8443)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Listening for WebRTC traffic at %d\n", 8443)

    // SettingEngineにUDPMuxを設定
    settingEngine.SetICEUDPMux(mux)

    // 共有設定でAPIを作成（グローバル変数として保持）
    api = webrtc.NewAPI(webrtc.WithSettingEngine(settingEngine))

    // HTTPサーバー起動
    http.Handle("/", http.FileServer(http.Dir(".")))
    http.HandleFunc("/doSignaling", doSignaling)
    panic(http.ListenAndServe(":8080", nil))
}
```

### Go側 - 各リクエストでPeerConnection作成

```go
func doSignaling(res http.ResponseWriter, req *http.Request) {
    // 共有APIから新しいPeerConnectionを作成
    // 全てのPeerConnectionが同じUDPMux（ポート8443）を使用
    peerConnection, err := api.NewPeerConnection(webrtc.Configuration{})
    if err != nil {
        panic(err)
    }

    peerConnection.OnICEConnectionStateChange(func(state webrtc.ICEConnectionState) {
        fmt.Printf("ICE Connection State has changed: %s\n", state.String())
    })

    peerConnection.OnDataChannel(func(d *webrtc.DataChannel) {
        d.OnOpen(func() {
            for range time.Tick(time.Second * 3) {
                d.SendText(time.Now().String())
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

### ブラウザ側 - 10個のPeerConnectionを作成（index.html）

```javascript
let createPeerConnection = () => {
  let pc = new RTCPeerConnection()
  let dc = pc.createDataChannel('data')

  dc.onopen = () => {
    // 選択されたICE候補ペアを表示
    let selectedPair = pc.sctp.transport.iceTransport.getSelectedCandidatePair()
    console.log('Local:', selectedPair.local.candidate)   // 各接続で異なるポート
    console.log('Remote:', selectedPair.remote.candidate) // 全て同じ8443
  }

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
}

// 10個のPeerConnectionを作成
for (i = 0; i < 10; i++) {
  createPeerConnection()
}
```

## 重要なAPI

### ice.NewMultiUDPMuxFromPort

```go
mux, err := ice.NewMultiUDPMuxFromPort(8443)
```

指定ポートでUDPソケットを作成し、複数のPeerConnectionで共有可能なMuxを返す。

### settingEngine.SetICEUDPMux

```go
settingEngine.SetICEUDPMux(mux)
```

SettingEngineにUDPMuxを設定。このSettingEngineから作成されたAPIを使う全てのPeerConnectionが同じポートを共有する。

## デモの動作

ブラウザで http://localhost:8080 を開くと:
- 10個のPeerConnectionが同時に作成される
- **Local（ブラウザ側）**: 各接続で異なるポート（例: 50001, 50002, ...）
- **Remote（Go側）**: 全て同じポート 8443

## ice-restart との比較

| 項目 | ice-single-port | ice-restart |
|------|-----------------|-------------|
| **目的** | 複数接続のポート共有 | ネットワーク変更時の接続維持 |
| **PeerConnection** | 毎回新規作成 | 同一インスタンスを再利用 |
| **ポート** | 単一ポート(8443)を共有 | 通常動作（ランダムポート） |
| **接続数** | 10個同時 | 1個 |
| **主要API** | `SetICEUDPMux()` | `createOffer({iceRestart: true})` |
| **ユースケース** | サーバーのスケーラビリティ | モバイル/WiFi切り替え |

### 解決する課題

| サンプル | 課題 |
|----------|------|
| **ice-single-port** | 「1000接続あったら1000ポート必要？」→ 1ポートで全て処理可能 |
| **ice-restart** | 「ネットワーク変わったら再接続？」→ 既存接続を維持したまま更新 |

### コード上の違い

```go
// ice-single-port: APIを共有、PeerConnectionは都度作成
var api *webrtc.API  // グローバルで共有
func doSignaling() {
    peerConnection, _ := api.NewPeerConnection(config)  // 毎回新規
}

// ice-restart: PeerConnectionを再利用
var peerConnection *webrtc.PeerConnection  // グローバルで保持
func doSignaling() {
    if peerConnection == nil {
        peerConnection, _ = webrtc.NewPeerConnection(config)  // 初回のみ
    }
    // 以降は同じインスタンスを使い続ける
}
```

## ユースケース

- **SFU/MCUサーバー**: 多数のクライアント接続を単一ポートで管理
- **ファイアウォール制限**: 開放ポート数を最小化
- **ロードバランサー**: 単一ポートへの振り分けが容易
- **コンテナ/Kubernetes**: ポートマッピングの簡素化
- **大規模配信**: 数千接続を効率的に処理

## 実行方法

```bash
cd examples/ice-single-port
go run *.go
```

ブラウザで http://localhost:8080 を開く。

## 注意事項

- UDPMuxはサーバー起動時に1回だけ作成する
- 全てのPeerConnectionは同じAPIインスタンスから作成する必要がある
- ポート8443はWebRTCトラフィック専用、HTTPサーバーは別ポート（8080）で動作
