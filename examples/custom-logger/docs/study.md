# custom-logger

このサンプルは **Pion WebRTCのログ出力をカスタマイズする方法** を示しています。

## 概要

デフォルトではPionは全てのログを`stdout`に出力しますが、このサンプルでは独自のロガーを注入して、ICE、DTLS、SCTPなど各サブシステムのログを自由にハンドリングできます。

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    Pion WebRTC                               │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │   ICE   │  │  DTLS   │  │  SCTP   │  │  DataChannel    │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────────┬────────┘  │
│       │            │            │                │           │
│       └────────────┴────────────┴────────────────┘           │
│                           │                                  │
│                           ▼                                  │
│                  ┌─────────────────┐                         │
│                  │ LoggerFactory   │ ← カスタム実装を注入     │
│                  └────────┬────────┘                         │
│                           │                                  │
└───────────────────────────┼──────────────────────────────────┘
                            ▼
                   ┌─────────────────┐
                   │  customLogger   │
                   │                 │
                   │  - Debug()      │
                   │  - Info()       │
                   │  - Warn()       │
                   │  - Error()      │
                   └─────────────────┘
                            │
                            ▼
                   ファイル、DB、監視システムなど
```

## 実行方法

### 準備

```bash
cd examples/custom-logger
```

### 実行

```bash
go run main.go
```

出力例:
```
Creating logger for ice
Creating logger for dtls
Peer Connection State has changed: connected (answerer)
Peer Connection State has changed: connected (offerer)
customLogger Debug: Adding a new peer-reflexive candidate: 10.8.21.1:51196
```

**注意**: このサンプルはブラウザ不要です。2つのPeerConnectionをローカルで作成し、互いに接続します。

## コード詳細

### 実装する2つのインターフェース

カスタムロガーを作成するには、以下の2つのインターフェースを実装します。

#### 1. LeveledLogger（ロガー本体）

```go
// logging.LeveledLogger インターフェースを満たす
type customLogger struct{}

// Traceレベルは無視（フィルタリング）
func (c customLogger) Trace(string)          {}
func (c customLogger) Tracef(string, ...any) {}

// Debug以上のレベルはカスタムフォーマットで出力
func (c customLogger) Debug(msg string) {
    fmt.Printf("customLogger Debug: %s\n", msg)
}
func (c customLogger) Debugf(format string, args ...any) {
    c.Debug(fmt.Sprintf(format, args...))
}

func (c customLogger) Info(msg string) {
    fmt.Printf("customLogger Info: %s\n", msg)
}
func (c customLogger) Infof(format string, args ...any) {
    c.Info(fmt.Sprintf(format, args...))
}

func (c customLogger) Warn(msg string) {
    fmt.Printf("customLogger Warn: %s\n", msg)
}
func (c customLogger) Warnf(format string, args ...any) {
    c.Warn(fmt.Sprintf(format, args...))
}

func (c customLogger) Error(msg string) {
    fmt.Printf("customLogger Error: %s\n", msg)
}
func (c customLogger) Errorf(format string, args ...any) {
    c.Error(fmt.Sprintf(format, args...))
}
```

#### 2. LoggerFactory（ロガー生成ファクトリ）

```go
// logging.LoggerFactory インターフェースを満たす
type customLoggerFactory struct{}

func (c customLoggerFactory) NewLogger(subsystem string) logging.LeveledLogger {
    fmt.Printf("Creating logger for %s\n", subsystem)
    return customLogger{}
}
```

**subsystem（サブシステム）の例**:
- `ice` - ICE候補の収集・接続
- `dtls` - DTLSハンドシェイク
- `sctp` - SCTPトランスポート
- `datachannel` - DataChannel

### カスタムロガーの注入

```go
// SettingEngineにLoggerFactoryを設定
s := webrtc.SettingEngine{
    LoggerFactory: customLoggerFactory{},
}

// カスタム設定を使用してAPIを作成
api := webrtc.NewAPI(webrtc.WithSettingEngine(s))

// このAPIで作成したPeerConnectionはカスタムロガーを使用
peerConnection, err := api.NewPeerConnection(config)
```

### ローカルでの接続テスト

```go
// 2つのPeerConnectionを作成
offerPeerConnection, _ := api.NewPeerConnection(config)
answerPeerConnection, _ := api.NewPeerConnection(config)

// ICE候補を相互に交換
offerPeerConnection.OnICECandidate(func(candidate *webrtc.ICECandidate) {
    if candidate != nil {
        answerPeerConnection.AddICECandidate(candidate.ToJSON())
    }
})
answerPeerConnection.OnICECandidate(func(candidate *webrtc.ICECandidate) {
    if candidate != nil {
        offerPeerConnection.AddICECandidate(candidate.ToJSON())
    }
})

// SDP交換
offer, _ := offerPeerConnection.CreateOffer(nil)
offerPeerConnection.SetLocalDescription(offer)
answerPeerConnection.SetRemoteDescription(offer)

answer, _ := answerPeerConnection.CreateAnswer(nil)
answerPeerConnection.SetLocalDescription(answer)
offerPeerConnection.SetRemoteDescription(answer)
```

## ログレベル

| レベル | 用途 | このサンプルでの扱い |
|--------|------|---------------------|
| Trace | 最も詳細なログ | 無視（出力しない） |
| Debug | デバッグ情報 | 出力 |
| Info | 一般情報 | 出力 |
| Warn | 警告 | 出力 |
| Error | エラー | 出力 |

## ユースケース

| 用途 | 説明 |
|------|------|
| **外部監視システム連携** | Datadog、Prometheus、CloudWatch等への送信 |
| **ファイル/DB保存** | ログの永続化、後からの分析 |
| **ログレベルフィルタ** | Trace無効化、本番環境ではError のみ出力など |
| **構造化ログ** | JSON形式での出力（ログ解析ツール連携） |
| **サブシステム別処理** | ICEログのみファイル保存など |
| **デバッグ** | 複雑なWebRTCフローの追跡 |

## 応用例

### サブシステム別のロガー

```go
func (c customLoggerFactory) NewLogger(subsystem string) logging.LeveledLogger {
    switch subsystem {
    case "ice":
        return iceLogger{}      // ICE専用ロガー
    case "dtls":
        return dtlsLogger{}     // DTLS専用ロガー
    default:
        return defaultLogger{}  // デフォルトロガー
    }
}
```

### JSON形式での出力

```go
func (c customLogger) Info(msg string) {
    log := map[string]string{
        "level":     "info",
        "message":   msg,
        "timestamp": time.Now().Format(time.RFC3339),
    }
    json.NewEncoder(os.Stdout).Encode(log)
}
```

### ファイルへの出力

```go
type fileLogger struct {
    file *os.File
}

func (f fileLogger) Info(msg string) {
    fmt.Fprintf(f.file, "[INFO] %s: %s\n", time.Now().Format(time.RFC3339), msg)
}
```

## 重要なAPI

| API | 説明 |
|-----|------|
| `logging.LeveledLogger` | ロガーインターフェース |
| `logging.LoggerFactory` | ロガーファクトリインターフェース |
| `webrtc.SettingEngine` | WebRTC設定のカスタマイズ |
| `webrtc.NewAPI()` | カスタム設定でAPIを作成 |

## 注意点

- `webrtc.NewPeerConnection()`ではなく`api.NewPeerConnection()`を使用
- サブシステムごとに`NewLogger()`が呼ばれる
- Traceレベルを無効化することでログ量を大幅に削減可能
- 本番環境ではログレベルを適切に設定することを推奨

## 関連サンプル

- **pion-to-pion**: ローカルでの2つのPeerConnection接続
- **data-channels**: DataChannelの使用例
