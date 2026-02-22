# ortc

このサンプルは **ORTC (Object Real-Time Communications) API** を使用してWebRTC接続を確立する方法を示しています。

## 概要

ORTCはWebRTCの低レベルAPIで、SDPを使用せずに各コンポーネント（ICE、DTLS、SCTP）を個別に操作できます。これにより、任意のシグナリングプロトコルを実装できます。

## WebRTC vs ORTC

| 項目 | WebRTC | ORTC |
|------|--------|------|
| シグナリング | SDP（Session Description Protocol） | 低レベルAPIで自由に実装 |
| 抽象度 | 高い（PeerConnection） | 低い（個別コンポーネント） |
| 柔軟性 | SDPに縛られる | 任意のプロトコルでシグナリング可能 |
| 使いやすさ | 簡単 | 複雑だが細かい制御が可能 |

## アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│                    ORTC                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │    ICE    │→ │   DTLS    │→ │   SCTP    │   │
│  │ Gatherer  │  │ Transport │  │ Transport │   │
│  │ Transport │  │           │  │           │   │
│  └───────────┘  └───────────┘  └───────────┘   │
│                                      ↓          │
│                              ┌───────────┐      │
│                              │DataChannel│      │
│                              └───────────┘      │
└─────────────────────────────────────────────────┘
```

### コンポーネントの役割

| コンポーネント | 役割 |
|----------------|------|
| ICEGatherer | ICE候補を収集 |
| ICETransport | ICE接続を管理 |
| DTLSTransport | 暗号化を提供 |
| SCTPTransport | 信頼性のあるデータ転送 |
| DataChannel | アプリケーションデータの送受信 |

## シグナリング情報（SDPの代わり）

```go
type Signal struct {
    ICECandidates    []webrtc.ICECandidate   // ICE候補
    ICEParameters    webrtc.ICEParameters    // uFrag, uPwd
    DTLSParameters   webrtc.DTLSParameters   // フィンガープリント
    SCTPCapabilities webrtc.SCTPCapabilities // 最大メッセージサイズ
}
```

SDPの代わりにJSON形式で必要な情報を交換します。

## 処理フロー

```
1. ICEGatherer作成 → ICE候補を収集
2. ICETransport作成
3. DTLSTransport作成
4. SCTPTransport作成
5. シグナリング情報を交換（JSON形式）
6. 各Transportを順番にStart
7. DataChannelで通信
```

## 実行方法

2つのターミナルを使用して、Go同士で通信します。

### 準備

```bash
cd examples/ortc
```

### ターミナル1: Offer側を起動

```bash
go run *.go -offer
```

出力例:
```
eyJpY2VDYW5kaWRhdGVzIjpbeyJmb3VuZGF0aW9uIjoiMTY4MDQ1NDY0...   ← これをコピー
```

### ターミナル2: Answer側を起動

```bash
echo "eyJpY2VDYW5kaWRhdGVzIjpbeyJmb3VuZGF0aW9uIjoiMTY4MDQ1NDY0..." | go run *.go
```

出力例:
```
eyJpY2VDYW5kaWRhdGVzIjpbeyJmb3VuZGF0aW9uIjoiMTIzNDU2Nzg5...   ← これをコピー
```

### ターミナル3（または別のターミナル）: AnswerをOffer側に送信

```bash
curl localhost:8080 -d "eyJpY2VDYW5kaWRhdGVzIjpbeyJmb3VuZGF0aW9uIjoiMTIzNDU2Nzg5..."
```

### 成功時の出力

両方のターミナルで以下が表示されます:

```
Data channel 'Foo'-'1' open. Random messages will now be sent to any connected DataChannels every 5 seconds
Sending AbCdEfGhIjKlMnO
Message from DataChannel 'Foo': 'XyZaBcDeFgHiJkL'
Sending PqRsTuVwXyZaBcD
...
```

## クイックスタート（まとめ）

```bash
cd examples/ortc

# ターミナル1: Offer側
go run *.go -offer
# → base64文字列をコピー

# ターミナル2: Answer側（コピーしたbase64を渡す）
echo "{ターミナル1のbase64}" | go run *.go
# → 別のbase64文字列をコピー

# ターミナル3: AnswerをOffer側に送信
curl localhost:8080 -d "{ターミナル2のbase64}"

# → 両方のターミナルでDataChannel通信が開始
```

## コード詳細

### ICEGathererの作成と候補収集

```go
// ICE収集オプション
iceOptions := webrtc.ICEGatherOptions{
    ICEServers: []webrtc.ICEServer{
        {URLs: []string{"stun:stun.l.google.com:19302"}},
    },
}

// ICEGathererを作成
api := webrtc.NewAPI()
gatherer, _ := api.NewICEGatherer(iceOptions)

// 収集完了を待機
gatherFinished := make(chan struct{})
gatherer.OnLocalCandidate(func(candidate *webrtc.ICECandidate) {
    if candidate == nil {
        close(gatherFinished)  // 収集完了
    }
})

// 候補を収集
gatherer.Gather()
<-gatherFinished

// 収集した候補とパラメータを取得
iceCandidates, _ := gatherer.GetLocalCandidates()
iceParams, _ := gatherer.GetLocalParameters()
```

### Transportの作成

```go
// ICETransportを作成
ice := api.NewICETransport(gatherer)

// DTLSTransportを作成（ICEの上に構築）
dtls, _ := api.NewDTLSTransport(ice, nil)

// SCTPTransportを作成（DTLSの上に構築）
sctp := api.NewSCTPTransport(dtls)
```

### シグナリング情報の作成と交換

```go
// ローカル情報を収集
dtlsParams, _ := dtls.GetLocalParameters()
sctpCapabilities := sctp.GetCapabilities()

// シグナリング情報を作成
signal := Signal{
    ICECandidates:    iceCandidates,
    ICEParameters:    iceParams,
    DTLSParameters:   dtlsParams,
    SCTPCapabilities: sctpCapabilities,
}

// base64エンコードして出力
fmt.Println(encode(signal))
```

### リモート情報の設定とTransportの開始

```go
// リモートのICE候補を設定
ice.SetRemoteCandidates(remoteSignal.ICECandidates)

// ICERoleを設定（OfferはControlling、AnswerはControlled）
iceRole := webrtc.ICERoleControlled
if isOffer {
    iceRole = webrtc.ICERoleControlling
}

// 各Transportを順番に開始
ice.Start(nil, remoteSignal.ICEParameters, &iceRole)
dtls.Start(remoteSignal.DTLSParameters)
sctp.Start(remoteSignal.SCTPCapabilities)
```

### DataChannelの作成（Offer側のみ）

```go
if isOffer {
    var id uint16 = 1
    dcParams := &webrtc.DataChannelParameters{
        Label: "Foo",
        ID:    &id,
    }
    channel, _ := api.NewDataChannel(sctp, dcParams)

    // メッセージ送受信ハンドラを設定
    go handleOnOpen(channel)()
    channel.OnMessage(func(msg webrtc.DataChannelMessage) {
        fmt.Printf("Message: %s\n", string(msg.Data))
    })
}
```

### DataChannelの受信（Answer側）

```go
sctp.OnDataChannel(func(channel *webrtc.DataChannel) {
    fmt.Printf("New DataChannel %s %d\n", channel.Label(), channel.ID())

    channel.OnOpen(handleOnOpen(channel))
    channel.OnMessage(func(msg webrtc.DataChannelMessage) {
        fmt.Printf("Message: %s\n", string(msg.Data))
    })
})
```

## PeerConnectionとの比較

```go
// WebRTC（PeerConnection）- 高レベルAPI
pc, _ := webrtc.NewPeerConnection(config)
pc.CreateOffer()  // 内部でICE/DTLS/SCTPを自動管理
dc, _ := pc.CreateDataChannel("data", nil)

// ORTC（低レベルAPI）- 各コンポーネントを手動管理
gatherer, _ := api.NewICEGatherer(options)
ice := api.NewICETransport(gatherer)
dtls, _ := api.NewDTLSTransport(ice, nil)
sctp := api.NewSCTPTransport(dtls)
dc, _ := api.NewDataChannel(sctp, params)

// 各コンポーネントを手動で接続・開始
ice.Start(...)
dtls.Start(...)
sctp.Start(...)
```

## ユースケース

- **カスタムシグナリング**: SDPではなく独自プロトコルを使いたい場合
- **より細かい制御**: 各コンポーネントを個別に操作したい場合
- **学習目的**: WebRTCの内部構造を理解する
- **特殊な要件**: SDPでは表現できない設定が必要な場合

## 注意点

- 3つのターミナル（または2つ+curl）が必要
- base64文字列は長いので、コピー&ペーストは慎重に
- Offer側はポート8080でHTTPサーバーを起動して待機
- ICERoleの設定が重要（OfferはControlling、AnswerはControlled）

## 関連サンプル

- **data-channels**: PeerConnectionを使った通常のDataChannel通信
- **ortc-media**: ORTCでメディア（動画/音声）を扱う場合
