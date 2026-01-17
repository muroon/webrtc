# Pion WebRTC

Go言語で実装されたWebRTC APIの純粋なGo実装です。

## 概要

- **バージョン**: v4.x
- **特徴**: CGo（C言語バインディング）を使用しない純粋なGo実装
- **対応プラットフォーム**: Windows, macOS, Linux, FreeBSD, iOS, Android, WebAssembly

## 主な機能

### PeerConnection API
- W3C WebRTC仕様の完全実装
- DataChannels（双方向データ転送）
- 音声・映像の送受信
- 再ネゴシエーション
- Plan-BとUnified Plan両対応

### 接続機能
- フルICEエージェント（STUN/TURN対応）
- ICE Restart、Trickle ICE
- mDNS候補

### メディア
- RTP/RTCPへの直接アクセス
- Opus, PCM, H264, VP8, VP9などのコーデック対応
- Simulcast、NACK、帯域推定

## アーキテクチャ

```
PeerConnection
    ├── ICETransport (接続確立)
    │       └── pion/ice
    ├── DTLSTransport (暗号化)
    │       └── pion/dtls
    ├── SCTPTransport (DataChannel用)
    │       └── pion/sctp
    ├── RTPTransceiver (メディア送受信)
    │       ├── RTPSender
    │       └── RTPReceiver
    └── DataChannel
```

## 主要コンポーネント

| ファイル | 役割 |
|---------|------|
| `api.go` | PeerConnection生成のファクトリ |
| `peerconnection.go` | WebRTC接続の中心オブジェクト |
| `mediaengine.go` | コーデックの登録・ネゴシエーション |
| `settingengine.go` | Pion固有の拡張設定 |
| `interceptor.go` | RTP/RTCP処理パイプライン |

## インストール

```bash
go get github.com/pion/webrtc/v4
```

## 基本的な使い方

```go
package main

import (
    "github.com/pion/webrtc/v4"
)

func main() {
    // PeerConnectionの設定
    config := webrtc.Configuration{
        ICEServers: []webrtc.ICEServer{
            {URLs: []string{"stun:stun.l.google.com:19302"}},
        },
    }

    // PeerConnectionの作成
    peerConnection, err := webrtc.NewPeerConnection(config)
    if err != nil {
        panic(err)
    }
    defer peerConnection.Close()

    // DataChannelの作成
    dataChannel, err := peerConnection.CreateDataChannel("data", nil)
    if err != nil {
        panic(err)
    }

    // DataChannelのイベントハンドラ
    dataChannel.OnOpen(func() {
        dataChannel.SendText("Hello, World!")
    })

    dataChannel.OnMessage(func(msg webrtc.DataChannelMessage) {
        println("受信:", string(msg.Data))
    })
}
```

## サンプル

`examples/`ディレクトリに多数のサンプルがあります：

### メディアAPI
- `play-from-disk`: ファイルからブラウザへ動画配信
- `save-to-disk`: Webカメラ映像をサーバーに保存
- `broadcast`: 1対多の動画配信
- `simulcast`: Simulcastストリームの受信とデマルチプレクス
- `rtp-forwarder`: RTPによる音声・映像転送

### DataChannel API
- `data-channels`: DataChannelでのメッセージ送受信
- `data-channels-detach`: 低レベルDataChannel API
- `pion-to-pion`: Pion同士の直接通信

### その他
- `ice-restart`: ネットワーク間のローミング
- `ice-single-port`: 単一ポートでの複数接続
- `trickle-ice`: Trickle ICE API

### サンプルの実行

```bash
# サンプルサーバーの起動
cd examples
go run examples.go

# ブラウザで http://localhost にアクセス
```

## 開発

### テスト

```bash
# 全テスト実行
go test ./...

# 特定のテスト実行
go test -run TestFunctionName ./...

# レースコンディション検出付き
go test -race ./...
```

### ビルド

```bash
# ライブラリのビルド
go build ./...

# サンプルのビルド
go build ./examples/play-from-disk/

# WebAssembly向けビルド
GOOS=js GOARCH=wasm go build -o demo.wasm ./examples/data-channels/jsfiddle/
```

## パッケージ構成

```
.
├── *.go                    # メインWebRTC API
├── internal/
│   ├── fmtp/              # フォーマットパラメータ解析（H264, VP9, AV1）
│   ├── mux/               # プロトコル多重化
│   └── util/              # ユーティリティ
├── pkg/
│   ├── media/             # メディアファイル読み書き
│   │   ├── ivfreader/     # IVFファイル読み込み
│   │   ├── ivfwriter/     # IVFファイル書き込み
│   │   ├── oggreader/     # OGGファイル読み込み
│   │   ├── oggwriter/     # OGGファイル書き込み
│   │   ├── h264reader/    # H264ファイル読み込み
│   │   └── h264writer/    # H264ファイル書き込み
│   └── rtcerr/            # WebRTCエラー型
└── examples/              # サンプルコード
```

## 依存ライブラリ

Pion WebRTCは以下のPionライブラリで構成されています：

| ライブラリ | 役割 |
|-----------|------|
| `pion/ice` | ICEエージェント実装 |
| `pion/dtls` | DTLS 1.2実装 |
| `pion/srtp` | SRTP暗号化 |
| `pion/sctp` | SCTPプロトコル |
| `pion/sdp` | SDP解析 |
| `pion/rtp` | RTPパケット処理 |
| `pion/rtcp` | RTCPパケット処理 |
| `pion/interceptor` | RTP/RTCP処理パイプライン |

## リンク

- [GitHub](https://github.com/pion/webrtc)
- [GoDoc](https://pkg.go.dev/github.com/pion/webrtc/v4)
- [Discord](https://discord.gg/PngbdqpFbt)
- [WebRTC for the Curious](https://webrtcforthecurious.com) - WebRTCの詳細な解説書

## ライセンス

MIT License
