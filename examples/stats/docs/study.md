# stats

このサンプルは **WebRTC統計情報（webrtc-stats）を取得・表示する方法** を示しています。

## 概要

[webrtc-stats](https://www.w3.org/TR/webrtc-stats/) APIを使用して、PeerConnectionの統計情報にアクセスします。セッション中に何が起きているかを把握し、通信品質の分析やデバッグに役立てることができます。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  Webcam ────────┼── WebRTC ──────────►│  OnTrack        │
│  Microphone ────┼──────────────────►  │      │          │
│                 │                     │      ▼          │
└─────────────────┘                     │  statsGetter    │
                                        │      │          │
                                        │      ▼          │
                                        │  統計情報を出力  │
                                        │                 │
                                        │  - パケット数    │
                                        │  - ロス率       │
                                        │  - ジッター     │
                                        │  - バイト数     │
                                        └─────────────────┘
```

## 取得できる統計情報

### InboundRTPStreamStats（受信RTPストリーム）

| フィールド | 説明 |
|-----------|------|
| PacketsReceived | 受信パケット数 |
| PacketsLost | 損失パケット数 |
| Jitter | ジッター（パケット到着間隔のばらつき） |
| LastPacketReceivedTimestamp | 最後のパケット受信時刻 |
| HeaderBytesReceived | 受信ヘッダーバイト数 |
| BytesReceived | 受信総バイト数 |
| FIRCount | FIR（Full Intra Request）送信回数 |
| PLICount | PLI（Picture Loss Indication）送信回数 |
| NACKCount | NACK（再送要求）送信回数 |

### ICECandidateStats（ICE候補）

| フィールド | 説明 |
|-----------|------|
| IP | リモートIPアドレス |
| Port | リモートポート番号 |
| Type | 候補タイプ（remote-candidate等） |

## 実行方法

### 準備

```bash
cd examples/stats
```

### 1. ブラウザでOfferを取得

1. [jsfiddle.net/s179hacu/](https://jsfiddle.net/s179hacu/) を開く
2. カメラ/マイクを許可
3. 「Copy browser SDP to clipboard」をクリック

### 2. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Connection State has changed checking
eyJ0eXBlIjoiYW5zd2VyIiwic2RwIjoiLi4uIn0=   ← これをコピー
```

### 3. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. 5秒ごとに統計情報が出力される

## クイックスタート（まとめ）

```bash
cd examples/stats

# 1. ブラウザで https://jsfiddle.net/s179hacu/ を開く
# 2. カメラ/マイクを許可
# 3. 「Copy browser SDP to clipboard」をクリック

# 4. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 5. 出力されたAnswerをブラウザに貼り付け
# 6. 「Start Session」をクリック
# → 5秒ごとに統計情報が出力される
```

## 出力例

```
New incoming track with codec: video/VP8
Stats for: video/VP8
InboundRTPStreamStats:
        PacketsReceived: 1255
        PacketsLost: 0
        Jitter: 588.9559641717999
        LastPacketReceivedTimestamp: 2023-04-26 13:16:16.63591134 -0400 EDT
        HeaderBytesReceived: 25100
        BytesReceived: 1361125
        FIRCount: 0
        PLICount: 0
        NACKCount: 0

remote-candidate IP(192.168.1.93) Port(59239)
remote-candidate IP(172.18.176.1) Port(59241)
remote-candidate IP(fd4d:d991:c340:6749:8c53:ee52:ae8c:14d4) Port(59238)
```

## コード詳細

### Statsインターセプターの設定

```go
import (
    "github.com/pion/interceptor"
    "github.com/pion/interceptor/pkg/stats"
)

// InterceptorRegistryを作成
interceptorRegistry := &interceptor.Registry{}

// Stats用のインターセプターを作成
statsInterceptorFactory, err := stats.NewInterceptor()

// 統計取得用のGetterを保存
var statsGetter stats.Getter
statsInterceptorFactory.OnNewPeerConnection(func(_ string, g stats.Getter) {
    statsGetter = g
})

// インターセプターを登録
interceptorRegistry.Add(statsInterceptorFactory)

// デフォルトのインターセプターも追加
webrtc.RegisterDefaultInterceptors(mediaEngine, interceptorRegistry)

// APIを作成
api := webrtc.NewAPI(
    webrtc.WithMediaEngine(mediaEngine),
    webrtc.WithInterceptorRegistry(interceptorRegistry),
)
```

### トラックごとの統計取得

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    fmt.Printf("New incoming track with codec: %s\n", track.Codec().MimeType)

    go func() {
        for {
            // SSRCを指定して統計を取得
            stats := statsGetter.Get(uint32(track.SSRC()))

            fmt.Printf("Stats for: %s\n", track.Codec().MimeType)
            fmt.Println(stats.InboundRTPStreamStats)

            time.Sleep(5 * time.Second)
        }
    }()

    // パケットを読み取り（必須）
    rtpBuff := make([]byte, 1500)
    for {
        track.Read(rtpBuff)
    }
})
```

### PeerConnection全体の統計取得

```go
for _, s := range peerConnection.GetStats() {
    switch stat := s.(type) {
    case webrtc.ICECandidateStats:
        if stat.Type == webrtc.StatsTypeRemoteCandidate {
            fmt.Printf("%s IP(%s) Port(%d)\n", stat.Type, stat.IP, stat.Port)
        }
    }
}
```

## 統計情報の解釈

### パケットロスの確認

```go
stats := statsGetter.Get(uint32(track.SSRC()))
lossRate := float64(stats.PacketsLost) / float64(stats.PacketsReceived) * 100
fmt.Printf("パケットロス率: %.2f%%\n", lossRate)
```

### ビットレートの計算

```go
// 前回の統計と比較して計算
prevBytes := previousStats.BytesReceived
currBytes := currentStats.BytesReceived
interval := 5.0 // 秒

bitrate := float64(currBytes-prevBytes) * 8 / interval / 1000 // kbps
fmt.Printf("ビットレート: %.2f kbps\n", bitrate)
```

### ジッターの評価

| ジッター値 | 品質 |
|-----------|------|
| < 30ms | 良好 |
| 30-50ms | 許容範囲 |
| > 50ms | 品質低下の可能性 |

## 重要なAPI

| API | 説明 |
|-----|------|
| `stats.NewInterceptor()` | Stats用インターセプターを作成 |
| `statsGetter.Get(ssrc)` | 指定SSRCの統計を取得 |
| `peerConnection.GetStats()` | PeerConnection全体の統計を取得 |
| `webrtc.ICECandidateStats` | ICE候補の統計情報 |
| `stats.InboundRTPStreamStats` | 受信RTPストリームの統計 |

## 統計の種類

| 統計タイプ | 説明 |
|-----------|------|
| InboundRTPStreamStats | 受信RTPストリームの統計 |
| OutboundRTPStreamStats | 送信RTPストリームの統計 |
| ICECandidateStats | ICE候補の情報 |
| ICECandidatePairStats | ICE候補ペアの情報 |
| TransportStats | トランスポート層の統計 |

## ユースケース

| 用途 | 説明 |
|------|------|
| **通信品質監視** | パケットロス、ジッターのリアルタイム監視 |
| **デバッグ** | 接続問題の原因特定 |
| **帯域推定** | 受信バイト数からビットレートを計算 |
| **品質レポート** | 通話終了後の品質サマリー作成 |
| **適応制御** | 品質に応じてビットレート調整 |
| **アラート** | パケットロス率が閾値を超えたら通知 |

## 注意点

- 統計情報は5秒間隔で取得（`statsInterval`で変更可能）
- トラックのRTPパケットは読み取る必要がある（破棄しても可）
- ICE接続がcheckingの間は統計を出力しない
- SSRCはトラックごとに一意の識別子

## 関連サンプル

- **rtcp-processing**: RTCPパケットの処理
- **custom-logger**: カスタムログ出力
- **reflect**: 映像のミラーリング
