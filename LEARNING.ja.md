# Pion WebRTCを理解するための知識

このドキュメントでは、Pion WebRTCを理解するために必要な知識をレベル別に整理しています。

## 基礎知識（必須）

### 1. Go言語
- 基本文法、インターフェース、構造体
- goroutineとチャネル（並行処理が多用される）
- `sync.Mutex`、`sync.RWMutex`、`atomic`パッケージ
- コンテキスト（`context.Context`）

### 2. ネットワークの基礎
- TCP/UDPの違い
- IPアドレス、ポート、NAT
- クライアント・サーバーモデル vs P2P

## WebRTC関連知識（重要）

### 3. WebRTCの基本概念

| 概念 | 説明 |
|------|------|
| **Signaling** | SDP（Session Description Protocol）の交換 |
| **Offer/Answer** | 接続ネゴシエーションモデル |
| **ICE** | NAT越えのための接続確立プロトコル |
| **STUN/TURN** | NAT越えサーバー |
| **Trickle ICE** | 候補を逐次送信する最適化 |

### 4. トランスポート層

| プロトコル | 用途 |
|-----------|------|
| **DTLS** | UDP上のTLS（暗号化） |
| **SRTP/SRTCP** | メディアの暗号化 |
| **SCTP** | DataChannelの信頼性制御 |

### 5. メディア関連

| 概念 | 説明 |
|------|------|
| **RTP** | リアルタイムメディア転送プロトコル |
| **RTCP** | RTPの制御プロトコル（統計、フィードバック） |
| **コーデック** | VP8, VP9, H264（映像）、Opus（音声） |
| **SDP** | メディア能力の記述フォーマット |

## 発展的な知識（深く理解するため）

### 6. WebRTC仕様
- [W3C WebRTC 仕様](https://www.w3.org/TR/webrtc/)
- PeerConnection状態マシン
- Unified Plan vs Plan-B

### 7. 品質制御

| 技術 | 目的 |
|------|------|
| **NACK** | パケットロス時の再送要求 |
| **PLI/FIR** | キーフレーム要求 |
| **TWCC** | 帯域推定（Transport Wide Congestion Control） |
| **Simulcast** | 複数品質ストリームの同時送信 |

### 8. 暗号技術
- TLS/DTLSハンドシェイク
- 証明書とフィンガープリント
- ECDSA、AES-GCM

## 学習リソース

### 書籍・ドキュメント

| リソース | 内容 |
|---------|------|
| [WebRTC for the Curious](https://webrtcforthecurious.com) | WebRTCの仕組みを深く解説（無料） |
| [High Performance Browser Networking](https://hpbn.co/) | WebRTCの章あり（無料） |
| [Pion GoDoc](https://pkg.go.dev/github.com/pion/webrtc/v4) | APIリファレンス |

### RFC（プロトコル仕様）

| RFC | 内容 |
|-----|------|
| RFC 8825 | WebRTC概要 |
| RFC 8445 | ICE |
| RFC 3550 | RTP |
| RFC 4566 | SDP |
| RFC 6347 | DTLS |

## 学習の順序（おすすめ）

```
1. Go言語の基礎
   ↓
2. WebRTC for the Curiousを読む
   ↓
3. Pionのexamplesを動かす
   - data-channels（シンプル）
   - play-from-disk（メディア）
   ↓
4. PeerConnectionのコードを読む
   ↓
5. ICE/DTLS/SCTPの各トランスポートを理解
   ↓
6. RFCを参照しながら詳細を理解
```

## このコードベースの特徴的なパターン

### 1. atomic.Valueによる状態管理

```go
state atomic.Value // ICEConnectionState
```

### 2. コールバックハンドラのパターン

```go
onTrackHandler func(*TrackRemote, *RTPReceiver)
```

### 3. 階層化されたトランスポート

```
ICETransport → DTLSTransport → SCTPTransport
```

### 4. ビルドタグによるプラットフォーム分岐

```go
//go:build !js  // ネイティブGo
//go:build js   // WebAssembly（ブラウザAPI利用）
```

## 補足

特に **WebRTC for the Curious** は、Pionの開発者が執筆に関わっており、このコードベースを理解する上で最も役立つリソースです。
