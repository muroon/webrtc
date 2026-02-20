# examples/data-channels/

WebRTCの **DataChannel** を使って、ブラウザとGoサーバー間でテキストメッセージを双方向に送受信するデモ。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `main.go` | Go側サーバー（ネイティブ実行） |
| `jsfiddle/demo.js` | ブラウザ側ロジック（JavaScript版） |
| `jsfiddle/main.go` | ブラウザ側ロジック（Wasm版、`GOOS=js`でビルド） |
| `jsfiddle/demo.html` | ブラウザUI |

## 動作の流れ

### 1. シグナリング（手動コピペ方式）

このサンプルにはシグナリングサーバーがなく、SDP（Session Description）をbase64エンコードして手動でコピペすることで接続を確立する。

1. ブラウザ側がOfferを作成し、base64文字列としてテキストエリアに表示
2. ユーザーがその文字列をコピーしてGo側のstdinに貼り付ける
3. Go側がOfferを受け取り、Answerを生成してbase64で標準出力に表示
4. ユーザーがAnswerをブラウザ側のテキストエリアに貼り付けて「Start Session」ボタンを押す

### 2. DataChannelでの通信

- ブラウザ側が `"foo"` という名前のDataChannelを作成
- 接続確立後、Go側は5秒ごとにランダムな15文字の英字メッセージを自動送信
- ブラウザ側はテキストエリアに入力したメッセージを「Send Message」ボタンで送信
- 双方とも受信メッセージをログに表示

## Go側 (`main.go`) のポイント

- **Answerside（受信側）** として動作。`OnDataChannel` でブラウザが作成したDataChannelを受け付ける
- ICE Gatheringが完了するまでブロックし、Trickle ICEは使わない（シグナリングが1回のやり取りで済むようにするため）
- `encode()`/`decode()` でSDPをbase64 JSONとしてやり取り

## ブラウザ側（`jsfiddle/`）のポイント

- **Offerside（発信側）** として動作。`createDataChannel("foo")` でチャネルを作成
- JavaScript版（`demo.js`）とWasm版（`main.go`）の2種類があり、どちらも同じ機能
- Wasm版は `syscall/js` パッケージを使ってDOM操作を行い、`js.Global().Set()` でブラウザのグローバル関数を登録している

## 実行方法

```bash
# Go側を起動
go run examples/data-channels/main.go

# ブラウザ側はexamplesサーバー経由で開く
cd examples && go run examples.go
```

ブラウザでページを開くとOfferのbase64文字列が表示されるので、それをGo側ターミナルに貼り付け、返されたAnswer文字列をブラウザに貼り付ければ接続が確立する。

## STUNサーバーの役割と通信の仕組み

### STUNサーバーは通信経路ではない

ブラウザ（JSFiddle）とローカルのGo側が同じSTUNサーバー（`stun:stun.l.google.com:19302`）を使っているが、**同じSTUNサーバーを使っているから通信できるわけではない**。

STUNサーバーはデータの中継は一切しない。役割は1つだけ：

> **自分のグローバルIP・ポートを教えてもらう**（NAT越しの自分のアドレスを知る）

```
ピア  ──「私のIPは？」──>  stun.l.google.com:19302
ピア  <──「203.0.113.5:54321だよ」──  stun.l.google.com:19302
```

これにより得られたアドレス（ICE Candidate）がSDPに含まれる。

### 通信が成立する流れ

```
1. ブラウザ → STUNで自分のグローバルIPを取得 → Offer(SDP)に含める
2. ユーザーがOffer(SDP)を手動コピペでGo側に渡す
3. Go側 → STUNで自分のグローバルIPを取得 → Answer(SDP)に含める
4. ユーザーがAnswer(SDP)を手動コピペでブラウザに渡す
5. 双方が相手のIPを知った状態でICE接続チェック → P2P直接通信開始
```

通信できる理由：

- **SDPの手動交換**によって、互いのIPアドレス・ポート情報を交換している
- **STUNは前準備**（自分のアドレスを知るため）に使われているだけ
- 同じSTUNサーバーでも別のSTUNサーバーでも、自分のグローバルIPさえ分かれば問題ない

### STUNサーバーが異なっていても通信できる

例えばブラウザが`stun:stun.l.google.com:19302`、Go側が`stun:stun1.l.google.com:19302`を使っても**問題なく通信できる**。どちらもNATの外側アドレスを教えてくれるだけなので、同じである必要はない。

### P2P直接通信ができない場合

NATの種類によってはSTUNだけでは相手に直接到達できないことがある。その場合は**TURNサーバー**（リレーサーバー）が必要になるが、このサンプルではTURNは設定されていない。

## トランシーバー（RTPTransceiver）について

このサンプルでは**トランシーバーは使われていない**。DataChannelのみを使用している。

### トランシーバーとDataChannelの違い

| 機能 | 用途 | プロトコル | このサンプル |
|---|---|---|---|
| **RTPTransceiver** | 音声・映像（メディア）の送受信 | RTP/SRTP | 未使用 |
| **DataChannel** | 任意のテキスト/バイナリデータの送受信 | SCTP | 使用 |

トランシーバーはRTP（Real-time Transport Protocol）を使ってメディアストリームを扱うためのもので、内部的に `RTPSender` + `RTPReceiver` のペアで構成される。

一方、DataChannelは **SCTP** プロトコル上で動作するため、RTPトランシーバーとは全く別の経路を使う。

### トランスポートスタックの違い

```
メディア通信（トランシーバー使用時）:
  ICE → DTLS → SRTP → RTPTransceiver（音声/映像）

DataChannel通信（このサンプル）:
  ICE → DTLS → SCTP → DataChannel（テキスト/バイナリ）
```

トランシーバーが使われているサンプルは `examples/play-from-disk/` や `examples/rtp-to-webrtc/` など、メディアを扱うサンプルを参照。
