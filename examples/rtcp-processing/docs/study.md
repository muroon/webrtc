# rtcp-processing

このサンプルは **RTCPパケットを受信・処理する方法** を示しています。

## 概要

RTCPパケットを読み取り、その内容をコンソールに出力します。このサンプルはRTPReceiverを使用した受信側の処理を示していますが、同様のAPIはRTPSender（送信側）にも存在します。

## RTCPとは？

**RTCP（RTP Control Protocol）** は、RTPと並行して使用される制御プロトコルです。メディアの品質情報や統計データを交換するために使用されます。

## アーキテクチャ

```
┌─────────────────┐                     ┌─────────────────┐
│    Browser      │                     │    Go Server    │
│                 │                     │                 │
│  RTP ──────────►│  メディアデータ       │                 │
│  (映像/音声)     │  (video/audio)      │                 │
│                 │                     │                 │
│  RTCP ─────────►│  統計・制御情報       │  receiver.      │
│  (SR, RR, etc.) │                     │  ReadRTCP()     │
└─────────────────┘                     └─────────────────┘
```

## RTCPパケットの種類

| パケット | 名称 | 用途 |
|----------|------|------|
| SR | Sender Report | 送信側の統計（送信パケット数、バイト数、NTPタイムスタンプ） |
| RR | Receiver Report | 受信側の統計（パケットロス、ジッター、遅延） |
| SDES | Source Description | 送信元情報（CNAME、NAME、EMAIL等） |
| BYE | Goodbye | セッション終了通知 |
| PLI | Picture Loss Indication | キーフレーム要求 |
| FIR | Full Intra Request | 完全なイントラフレーム要求 |
| NACK | Negative Acknowledgment | パケット再送要求 |
| REMB | Receiver Estimated Maximum Bitrate | 推定帯域幅の通知 |

## 実行方法

### 準備

```bash
cd examples/rtcp-processing
```

### 1. ブラウザでOfferを取得

1. [jsfiddle.net/zurq6j7x/](https://jsfiddle.net/zurq6j7x/) を開く
2. カメラ/マイクを許可
3. 「Copy browser SDP to clipboard」をクリック

### 2. Goプログラムを実行

```bash
echo "{コピーしたbase64文字列}" | go run *.go
```

出力例:
```
Connection State has changed checking
Connection State has changed connected
Track has started streamId(abc123) id(video0) rid()
Received RTCP Packet: SenderReport from 1234567890
  NTPTime: 2023-01-01 12:00:00
  RTPTime: 123456
  PacketCount: 1000
  OctetCount: 150000
...
```

### 3. Answerをブラウザに貼り付け

1. 出力されたAnswer（base64文字列）をコピー
2. jsfiddleの2番目のテキストエリアに貼り付け
3. 「Start Session」をクリック
4. RTCPパケットがコンソールに出力される

## クイックスタート（まとめ）

```bash
cd examples/rtcp-processing

# 1. ブラウザで https://jsfiddle.net/zurq6j7x/ を開く
# 2. カメラ/マイクを許可
# 3. 「Copy browser SDP to clipboard」をクリック

# 4. 実行
echo "{コピーしたSDPのbase64文字列}" | go run *.go

# 5. 出力されたAnswerをブラウザに貼り付け
# 6. 「Start Session」をクリック
# → RTCPパケットがコンソールに出力される
```

## コード詳細

### RTCPパケットの読み取り

```go
peerConnection.OnTrack(func(track *webrtc.TrackRemote, receiver *webrtc.RTPReceiver) {
    fmt.Printf("Track has started streamId(%s) id(%s) rid(%s)\n",
        track.StreamID(), track.ID(), track.RID())

    for {
        // RTCPパケットを読み取り
        rtcpPackets, _, rtcpErr := receiver.ReadRTCP()
        if rtcpErr != nil {
            panic(rtcpErr)
        }

        // 各パケットを処理
        for _, r := range rtcpPackets {
            // パケット内容を文字列で出力
            if stringer, canString := r.(fmt.Stringer); canString {
                fmt.Printf("Received RTCP Packet: %v", stringer.String())
            }
        }
    }
})
```

### パケットタイプごとの処理例

```go
import "github.com/pion/rtcp"

for _, packet := range rtcpPackets {
    switch p := packet.(type) {
    case *rtcp.SenderReport:
        fmt.Printf("SR: NTPTime=%v, PacketCount=%d\n",
            p.NTPTime, p.PacketCount)

    case *rtcp.ReceiverReport:
        for _, report := range p.Reports {
            fmt.Printf("RR: FractionLost=%d, TotalLost=%d, Jitter=%d\n",
                report.FractionLost, report.TotalLost, report.Jitter)
        }

    case *rtcp.PictureLossIndication:
        fmt.Printf("PLI: MediaSSRC=%d\n", p.MediaSSRC)

    case *rtcp.ReceiverEstimatedMaximumBitrate:
        fmt.Printf("REMB: Bitrate=%d\n", p.Bitrate)
    }
}
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `receiver.ReadRTCP()` | RTCPパケットを読み取り（ブロッキング） |
| `peerConnection.WriteRTCP()` | RTCPパケットを送信 |
| `rtcp.SenderReport` | 送信者レポート |
| `rtcp.ReceiverReport` | 受信者レポート |
| `rtcp.PictureLossIndication` | PLI（キーフレーム要求） |

## RTPとRTCPの違い

| 項目 | RTP | RTCP |
|------|-----|------|
| 用途 | メディアデータ転送 | 統計・制御 |
| 頻度 | 高い（毎フレーム） | 低い（数秒ごと） |
| データ | 映像/音声フレーム | 統計情報・制御メッセージ |
| API | `track.ReadRTP()` | `receiver.ReadRTCP()` |
| 帯域 | 大部分を占める | 全体の5%以下が目安 |

## RTCPの役割

### 1. 品質監視

```
ReceiverReport:
  - FractionLost: 直近のパケットロス率
  - TotalLost: 累積パケットロス数
  - Jitter: パケット到着間隔のばらつき
  - LastSR/DLSR: ラウンドトリップ時間の計算に使用
```

### 2. 帯域推定

```
REMB (Receiver Estimated Maximum Bitrate):
  - 受信側が推定した利用可能帯域を送信側に通知
  - 送信側はこれに基づいてビットレートを調整
```

### 3. キーフレーム要求

```
PLI (Picture Loss Indication):
  - パケットロスで映像が乱れた場合に使用
  - 送信側に新しいキーフレームを要求
```

## ユースケース

- **品質監視**: パケットロス率、ジッターの測定
- **帯域推定**: REMBパケットから利用可能帯域を把握
- **デバッグ**: メディア配信の問題調査
- **適応制御**: 品質に応じてビットレート調整
- **統計収集**: 通話品質レポートの作成

## 注意点

- このサンプルはRTPReceiver（受信側）のRTCP処理を示している
- 送信側（RTPSender）にも同様のAPIが存在する
- RTCPパケットの読み取りはブロッキング処理
- 本番環境ではRTCP情報を基に適応的な品質制御を実装する

## 関連サンプル

- **reflect**: 映像のミラーリング（RTCP処理を含む）
- **broadcast**: 1対多の配信
- **simulcast**: 複数品質の配信
