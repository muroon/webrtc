# vnet

**vnet（Virtual Network）** は Pionの仮想ネットワーク層で、**実際のネットワークを使わずにWebRTC通信をシミュレート**できます。

## 概要

本番環境で発生する問題（パケットロス、ジッター、遅延など）をシミュレートしてテストできます。ブラウザや実際のネットワークを使わずに、ローカルで完結するテストが可能です。

詳細なドキュメント: [pion/transport/vnet](https://github.com/pion/transport/tree/master/vnet#vnet)

## vnetでできること

| 機能 | 説明 |
|------|------|
| **ネットワークトポロジのシミュレート** | STUN/TURNサーバーが必要な状況を再現 |
| **パケットロス/ジッター/並び替え** | 悪条件下でのアプリ動作をテスト |
| **帯域使用量の計測** | アプリの総コストを見積もり |
| **ネットワーク監視** | 通過するパケットをフィルタ/計測 |
| **NAT/Firewallシミュレート** | 複雑なネットワーク構成の再現 |

## アーキテクチャ

```
+ - - - - - - - - - - - - - - - - - - - - - - - +
                      VNet
| +-------------------------------------------+ |
  |              wan:vnet.Router              |
| +---------+----------------------+----------+ |
            |                      |
| +---------+----------+ +---------+----------+ |
  | offerVNet:vnet.Net | |answerVNet:vnet.Net |
| |     (1.2.3.4)      | |     (1.2.3.5)      | |
| +---------+----------+ +---------+----------+ |
            |                      |
+ - - - - - + - - - - - - - - - - -+- - - - - - +
            |                      |
  +---------+----------+ +---------+----------+
  |offerPeerConnection | |answerPeerConnection|
  +--------------------+ +--------------------+
```

## 主要コンポーネント

| コンポーネント | 説明 |
|---------------|------|
| `vnet.Router` | 仮想ルーター（ネットワークの中心） |
| `vnet.Net` | 仮想ネットワークインターフェース |
| `ChunkFilter` | パケットのフィルタリング/監視 |
| `Chunk` | 仮想ネットワーク上のパケット |

## サンプル: show-network-usage

`vnet/show-network-usage/` は、仮想ネットワークを流れるパケット量をモニタリングするサンプルです。

### 実行方法

```bash
cd examples/vnet/show-network-usage
go run main.go
```

### 出力例

```
Peer Connection State has changed: connected (offerer)
Peer Connection State has changed: connected (answerer)
2024/01/01 12:00:00 inbound throughput : 1234.5 [Byte/s]
2024/01/01 12:00:00 outbound throughput: 567.8 [Byte/s]
2024/01/01 12:00:02 inbound throughput : 1198.0 [Byte/s]
2024/01/01 12:00:02 outbound throughput: 543.2 [Byte/s]
```

## コード詳細

### 1. 仮想ルーターの作成

```go
import "github.com/pion/transport/v4/vnet"

// CIDRでネットワーク範囲を指定
wan, err := vnet.NewRouter(&vnet.RouterConfig{
    CIDR:          "1.2.3.0/24",
    LoggerFactory: logging.NewDefaultLoggerFactory(),
})
```

### 2. トラフィックモニタリング（ChunkFilter）

```go
var inboundBytes int32
var outboundBytes int32

wan.AddChunkFilter(func(chunk vnet.Chunk) bool {
    netType := chunk.SourceAddr().Network()
    if netType == "udp" {
        // 宛先が1.2.3.4ならinbound（受信）
        dstAddr := chunk.DestinationAddr().String()
        host, _, _ := net.SplitHostPort(dstAddr)
        if host == "1.2.3.4" {
            atomic.AddInt32(&inboundBytes, int32(len(chunk.UserData())))
        }

        // 送信元が1.2.3.4ならoutbound（送信）
        srcAddr := chunk.SourceAddr().String()
        host, _, _ = net.SplitHostPort(srcAddr)
        if host == "1.2.3.4" {
            atomic.AddInt32(&outboundBytes, int32(len(chunk.UserData())))
        }
    }
    return true  // trueでパケットを通過させる
})
```

### 3. スループットの計測

```go
go func() {
    duration := 2 * time.Second
    for {
        time.Sleep(duration)

        // 読み取りとリセットを同時に行う
        inBytes := atomic.SwapInt32(&inboundBytes, 0)
        outBytes := atomic.SwapInt32(&outboundBytes, 0)

        // スループットを計算
        inboundThroughput := float64(inBytes) / duration.Seconds()
        outboundThroughput := float64(outBytes) / duration.Seconds()

        log.Printf("inbound throughput : %.01f [Byte/s]\n", inboundThroughput)
        log.Printf("outbound throughput: %.01f [Byte/s]\n", outboundThroughput)
    }
}()
```

### 4. 仮想ネットワークインターフェースの作成

```go
// Offerer用（IP: 1.2.3.4）
offerVNet, err := vnet.NewNet(&vnet.NetConfig{
    StaticIPs: []string{"1.2.3.4"},
})
wan.AddNet(offerVNet)  // ルーターに追加

// Answerer用（IP: 1.2.3.5）
answerVNet, err := vnet.NewNet(&vnet.NetConfig{
    StaticIPs: []string{"1.2.3.5"},
})
wan.AddNet(answerVNet)  // ルーターに追加
```

### 5. WebRTCへの注入

```go
// SettingEngineに仮想ネットワークを設定
offerSettingEngine := webrtc.SettingEngine{}
offerSettingEngine.SetNet(offerVNet)
offerAPI := webrtc.NewAPI(webrtc.WithSettingEngine(offerSettingEngine))

answerSettingEngine := webrtc.SettingEngine{}
answerSettingEngine.SetNet(answerVNet)
answerAPI := webrtc.NewAPI(webrtc.WithSettingEngine(answerSettingEngine))
```

### 6. 仮想ネットワークの開始とPeerConnection作成

```go
// 仮想ネットワークを開始
wan.Start()

// 仮想ネットワーク上でPeerConnectionを作成
offerPeerConnection, _ := offerAPI.NewPeerConnection(webrtc.Configuration{})
answerPeerConnection, _ := answerAPI.NewPeerConnection(webrtc.Configuration{})
```

### 7. DataChannelでのメッセージ送信

```go
// DataChannelを作成
offerDataChannel, _ := offerPeerConnection.CreateDataChannel("label", nil)

// 100msごとにメッセージを送信
offerDataChannel.OnOpen(func() {
    for {
        time.Sleep(100 * time.Millisecond)
        offerDataChannel.SendText("My DataChannel Message")
    }
})
```

## ChunkFilterの活用

ChunkFilterは様々な用途に使用できます：

### パケットドロップ（パケットロスのシミュレート）

```go
dropRate := 0.1  // 10%ドロップ

wan.AddChunkFilter(func(chunk vnet.Chunk) bool {
    if rand.Float64() < dropRate {
        return false  // パケットをドロップ
    }
    return true  // パケットを通過
})
```

### 特定IPのブロック

```go
wan.AddChunkFilter(func(chunk vnet.Chunk) bool {
    host, _, _ := net.SplitHostPort(chunk.DestinationAddr().String())
    if host == "1.2.3.100" {
        return false  // このIPへのパケットをブロック
    }
    return true
})
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `vnet.NewRouter()` | 仮想ルーターを作成 |
| `vnet.NewNet()` | 仮想ネットワークインターフェースを作成 |
| `router.AddNet()` | ネットワークをルーターに追加 |
| `router.AddChunkFilter()` | パケットフィルターを追加 |
| `router.Start()` | 仮想ネットワークを開始 |
| `settingEngine.SetNet()` | WebRTCに仮想ネットワークを注入 |
| `chunk.UserData()` | UDPペイロードを取得 |
| `chunk.SourceAddr()` | 送信元アドレスを取得 |
| `chunk.DestinationAddr()` | 宛先アドレスを取得 |

## ユースケース

| 用途 | 説明 |
|------|------|
| **単体テスト** | ネットワークなしでWebRTCをテスト |
| **負荷テスト** | パケットロス/遅延下での動作確認 |
| **帯域計測** | 実際の通信量を計測 |
| **CI/CD** | 自動テスト環境でのWebRTCテスト |
| **NAT/Firewallシミュレート** | 複雑なネットワーク構成の再現 |
| **回帰テスト** | 再現可能なネットワーク条件でのテスト |

## 特徴

| 特徴 | 説明 |
|------|------|
| ブラウザ不要 | 完全にローカルで動作 |
| 実ネットワーク不要 | 仮想ネットワーク上で通信 |
| 再現可能 | 同じ条件でテストを繰り返し可能 |
| 高速 | 実ネットワークの遅延なし |
| 柔軟 | 様々なネットワーク条件をシミュレート |

## 注意点

- `SettingEngine.SetNet()`で仮想ネットワークを注入する
- `router.Start()`を呼び出してから通信を開始する
- ChunkFilterで`false`を返すとパケットがドロップされる
- 複数のChunkFilterを追加可能（順番に実行される）

## 関連サンプル

- **pion-to-pion**: 2つのPeerConnectionの接続
- **custom-logger**: カスタムログ出力
- **stats**: 統計情報の取得
