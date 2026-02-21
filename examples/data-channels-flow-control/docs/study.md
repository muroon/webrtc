# data-channels-flow-control 学習メモ

## 概要

### 目的
DataChannelの**フロー制御（輻輳制御）API**の使い方を示すサンプルです。

### 主要API
- `BufferedAmount()` - 現在のバッファ量を取得
- `SetBufferedAmountLowThreshold(th)` - 閾値を設定
- `BufferedAmountLowThreshold()` - 閾値を取得
- `OnBufferedAmountLow(f)` - バッファが閾値以下になった時のコールバック

### なぜ必要か
`Send()`/`SendText()`は即座にreturnするが、データは内部バッファにキューイングされる。送信レートがネットワーク速度を超えると、バッファが無限に増大しメモリ枯渇の原因となる。このAPIでペーシング（送信速度調整）が可能。

### コードの構造
```
offerPC (送信側) ----data----> answerPC (受信側)
```

| 定数 | 値 |
|------|-----|
| `bufferedAmountLowThreshold` | 512 KB |
| `maxBufferedAmount` | 1 MB |

### DataChannel設定
- `Ordered: false` - 順序保証なし（高速）
- `MaxRetransmits: 0` - 再送なし（UDP的挙動）

---

## 実行方法

### コマンド

```bash
# examples/data-channels-flow-control/ ディレクトリで実行
cd examples/data-channels-flow-control
go run main.go
```

または、プロジェクトルートから:

```bash
go run ./examples/data-channels-flow-control/
```

### 動作内容

このサンプルは**単一プロセス内で2つのPeerConnectionを作成**し、自己完結で動作します。

1. `offerPC`（送信側）と`answerPC`（受信側）を生成
2. シグナリング（SDP交換）を内部で自動実行
3. DataChannel "data" を確立
4. 1024バイトパケットを可能な限り高速に送信開始
5. 1秒ごとにスループット（Mbps）を表示

### 出力例

```
Peer Connection State has changed: connecting (offerer)
Peer Connection State has changed: connecting (answerer)
Peer Connection State has changed: connected (offerer)
Peer Connection State has changed: connected (answerer)
OnOpen: data-xxx. Start sending a series of 1024-byte packets as fast as it can
OnOpen: data-xxx. Start receiving data
Throughput: 179.118 Mbps
Throughput: 203.545 Mbps
Throughput: 211.516 Mbps
...
```

### 終了方法

`Ctrl+C` で終了（無限ループで動作し続けるため）

### 特徴

- 外部サーバーやブラウザ不要
- フロー制御の効果をスループット値で確認可能
- ローカル環境で手軽にDataChannelのパフォーマンステストができる

---

## フロー制御しているのはmain.goのどの部分?

フロー制御は `createOfferer()` 関数内の以下の部分で行われています。

### 1. 閾値の定義 (`main.go:18-21`)
```go
const (
    bufferedAmountLowThreshold uint64 = 512 * 1024  // 512 KB
    maxBufferedAmount          uint64 = 1024 * 1024 // 1 MB
)
```

### 2. 送信制御チャネル (`main.go:59`)
```go
sendMoreCh := make(chan struct{}, 1)
```

### 3. 送信ループでのバッファ監視 (`main.go:72-80`)
```go
for {
    err2 := dataChannel.Send(buf)
    check(err2)

    if dataChannel.BufferedAmount() > maxBufferedAmount {
        // Wait until the bufferedAmount becomes lower than the threshold
        <-sendMoreCh  // ここでブロック（送信停止）
    }
}
```

### 4. 閾値の設定 (`main.go:85`)
```go
dataChannel.SetBufferedAmountLowThreshold(bufferedAmountLowThreshold)
```

### 5. バッファ低下時のコールバック (`main.go:88-93`)
```go
dataChannel.OnBufferedAmountLow(func() {
    select {
    case sendMoreCh <- struct{}{}:  // 送信再開シグナル
    default:
    }
})
```

### フロー図

```
送信ループ
    |
    +---> Send(1024 bytes)
    |
    +---> BufferedAmount() > 1MB ?
    |       |
    |       YES --> <-sendMoreCh でブロック（待機）
    |                   ^
    |                   |
    |       OnBufferedAmountLow() が発火
    |       (バッファが 512KB 以下になった時)
    |                   |
    |       sendMoreCh <- struct{}{} で送信再開
    |
    +---> 繰り返し
```

この仕組みにより、バッファが1MBを超えると送信を一時停止し、512KB以下になると再開することで、メモリ枯渇を防いでいます。
