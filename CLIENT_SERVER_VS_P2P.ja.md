# クライアント・サーバーモデル vs P2P

## 基本概念

### クライアント・サーバーモデル

```
┌────────┐         ┌────────┐
│Client A│────────→│        │
└────────┘         │ Server │
┌────────┐         │        │
│Client B│────────→│        │
└────────┘         └────────┘

- サーバーが中心
- クライアントはサーバーに接続
- 例: Webサイト、API、データベース
```

### P2P（Peer-to-Peer）

```
┌────────┐
│ Peer A │←──┐
└────────┘   │
     ↑       ↓
     │   ┌────────┐
     └──→│ Peer B │
         └────────┘

- サーバー不要（または最小限）
- ピア同士が直接通信
- 例: WebRTC、BitTorrent、ブロックチェーン
```

## 比較表

| 項目 | クライアント・サーバー | P2P |
|------|----------------------|-----|
| **構造** | 中央集権 | 分散 |
| **遅延** | サーバー経由で遅い | 直接通信で低遅延 |
| **帯域** | サーバーに集中 | 分散 |
| **障害耐性** | サーバー障害で全停止 | 一部障害でも継続可能 |
| **スケール** | サーバー増強が必要 | ピア増加で自然にスケール |
| **NAT越え** | 不要（サーバーはグローバルIP） | 必要（ICE/STUN/TURN） |

## WebRTCでの使い分け

```
┌─────────────────────────────────────────────────┐
│                  Signaling                       │
│            (クライアント・サーバー)                │
│                                                  │
│  ┌────────┐    ┌────────┐    ┌────────┐        │
│  │Client A│───→│ Server │←───│Client B│        │
│  └────────┘    └────────┘    └────────┘        │
│       │                            │            │
│       │    SDP/ICE候補の交換       │            │
│       └────────────────────────────┘            │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│              Media/Data (P2P)                    │
│                                                  │
│        ┌────────┐      ┌────────┐              │
│        │ Peer A │←────→│ Peer B │              │
│        └────────┘      └────────┘              │
│                                                  │
│           音声・映像・データの直接通信            │
└─────────────────────────────────────────────────┘
```

## 学習リソース

### 入門レベル

| リソース | 内容 |
|---------|------|
| [MDN: WebRTCの基礎](https://developer.mozilla.org/ja/docs/Web/API/WebRTC_API/Protocols) | WebRTCプロトコルの概要 |
| [ネットワークはなぜつながるのか](https://www.amazon.co.jp/dp/4822283119) | TCP/IPの基礎から解説（書籍） |
| [マスタリングTCP/IP 入門編](https://www.amazon.co.jp/dp/4274224473) | ネットワークの定番入門書 |

### 動画

| リソース | 内容 |
|---------|------|
| YouTube「P2P ネットワーク 仕組み」で検索 | 視覚的な解説が多数 |
| [Computerphile - NAT](https://www.youtube.com/results?search_query=computerphile+NAT) | NATの仕組み（英語） |

### 実践

```bash
# Pionのdata-channelsサンプルで体験
cd examples
go run examples.go

# ブラウザで http://localhost を開く
# → data-channels を選択
# → P2P通信を体験
```

## P2P特有の課題

| 課題 | 説明 | 解決策 |
|------|------|--------|
| **NAT越え** | プライベートIP同士は直接通信不可 | ICE, STUN, TURN |
| **ピア発見** | 相手のIPアドレスをどう知るか | Signalingサーバー |
| **ファイアウォール** | 受信接続がブロックされる | TURN（リレー） |

## 簡単な実験

### クライアント・サーバーを体験

```go
// server.go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from server!")
    })
    http.ListenAndServe(":8080", nil)
}
```

### P2Pを体験（Pionの最小例）

```go
// 2つのPeerConnectionを作成し、直接接続
pc1, _ := webrtc.NewPeerConnection(webrtc.Configuration{})
pc2, _ := webrtc.NewPeerConnection(webrtc.Configuration{})

// ICE候補の交換
pc1.OnICECandidate(func(c *webrtc.ICECandidate) {
    if c != nil {
        pc2.AddICECandidate(c.ToJSON())
    }
})
pc2.OnICECandidate(func(c *webrtc.ICECandidate) {
    if c != nil {
        pc1.AddICECandidate(c.ToJSON())
    }
})

// Offer/Answerの交換
offer, _ := pc1.CreateOffer(nil)
pc1.SetLocalDescription(offer)
pc2.SetRemoteDescription(offer)

answer, _ := pc2.CreateAnswer(nil)
pc2.SetLocalDescription(answer)
pc1.SetRemoteDescription(answer)
```

## 学習の順序

```
1. TCP/IPの基礎を理解
   ↓
2. HTTPでクライアント・サーバーを実装してみる
   ↓
3. NATの仕組みを理解
   ↓
4. WebRTCのSignalingフローを理解
   ↓
5. PionのP2Pサンプルを動かす
   ↓
6. ICE/STUN/TURNの役割を理解
```

## 補足

特にNATの仕組みを理解することが、P2Pの課題を理解する鍵になります。
