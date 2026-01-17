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

## examples/を活用した学習ガイド

### Phase 1: Signalingの理解

**目標**: Offer/Answer、SDP、ICE候補の交換を理解する

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 1 | `data-channels` | 手動Signaling、SDPの中身を見る |
| 2 | `pion-to-pion` | ブラウザなしで純粋なGo同士の接続 |
| 3 | `trickle-ice` | ICE候補の逐次交換 |

```bash
# まずdata-channelsで手動Signalingを体験
cd examples/data-channels && go run main.go

# 次にpion-to-pionでGoだけで完結する例を見る
cd examples/pion-to-pion
go run offer/main.go  # ターミナル1
go run answer/main.go # ターミナル2
```

### Phase 2: DataChannel API

**目標**: データ送受信の仕組みを理解する

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 4 | `data-channels-detach` | 低レベルAPI、io.Reader/Writer |
| 5 | `data-channels-flow-control` | バックプレッシャー、フロー制御 |
| 6 | `ortc` | ORTC API（PeerConnectionを使わない方法） |

### Phase 3: ICEの深堀り

**目標**: NAT越え、接続確立の詳細を理解する

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 7 | `ice-restart` | ネットワーク切り替え時の再接続 |
| 8 | `ice-single-port` | 本番環境向け：単一ポートで複数接続 |
| 9 | `ice-tcp` | UDP blocked環境への対応 |

### Phase 4: メディア基礎

**目標**: 音声・映像の送受信を理解する

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 10 | `play-from-disk` | ファイル→ブラウザへ動画送信 |
| 11 | `save-to-disk` | ブラウザ→サーバーへ録画保存 |
| 12 | `reflect` | 受信したメディアをそのまま返す |

```bash
# play-from-diskを試す（VP8動画ファイルが必要）
cd examples/play-from-disk
go run main.go
```

### Phase 5: メディア応用

**目標**: 実践的なメディア処理を理解する

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 13 | `broadcast` | 1対多配信（SFU的パターン） |
| 14 | `simulcast` | 複数品質ストリームの処理 |
| 15 | `swap-tracks` | 動的なトラック切り替え |
| 16 | `rtcp-processing` | RTCP統計情報の取得 |

### Phase 6: RTP/RTCP直接操作

**目標**: 低レベルのメディア処理を理解する

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 17 | `rtp-forwarder` | RTPパケットの転送 |
| 18 | `rtp-to-webrtc` | 外部RTPソースの取り込み |
| 19 | `insertable-streams` | E2E暗号化 |

### Phase 7: 本番環境向け

**目標**: 運用に必要な知識を得る

| 順序 | サンプル | 学べること |
|------|---------|-----------|
| 20 | `custom-logger` | ログのカスタマイズ |
| 21 | `stats` | 接続統計の取得 |
| 22 | `vnet` | ネットワークシミュレーション |

### 推奨する学習フロー

```
Phase 1 (Signaling)
    │
    ├─→ Phase 2 (DataChannel) ─→ チャットアプリなど作れる
    │
    └─→ Phase 3 (ICE) ─→ Phase 4 (メディア基礎)
                              │
                              ├─→ Phase 5 (メディア応用) ─→ 配信アプリなど作れる
                              │
                              └─→ Phase 6 (RTP/RTCP) ─→ SFU/MCU開発へ
                                        │
                                        └─→ Phase 7 (本番運用)
```

### 各サンプルで見るべきポイント

| Phase | 注目ファイル/コード |
|-------|-------------------|
| 1 | `main.go`の`CreateOffer`/`CreateAnswer`周辺 |
| 2 | `OnDataChannel`、`OnMessage`コールバック |
| 3 | `OnICECandidate`、`AddICECandidate` |
| 4 | `AddTrack`、`OnTrack`コールバック |
| 5 | `RTPSender`、`RTPReceiver`の操作 |
| 6 | `ReadRTP`、`WriteRTP`の使い方 |

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
