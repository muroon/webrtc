<h1 align="center">
  サンプル集
</h1>

一般的なユースケースをカバーする豊富なサンプル集を用意しています。これらのサンプルを修正・拡張して、すぐに開発を始めることができます。

サードパーティライブラリを使用したより本格的なサンプルは、**[example-webrtc-applications](https://github.com/pion/example-webrtc-applications)** リポジトリをご覧ください。

### 概要

#### Media API

| サンプル | 説明 |
|---------|------|
| [Reflect](reflect) | 受信したメディアをそのまま送り返すサンプル。同じPeerConnectionを使用します。 |
| [Play from Disk](play-from-disk) | ディスクに保存されたファイルからブラウザに動画を送信するサンプル。 |
| [Play from Disk Renegotiation](play-from-disk-renegotiation) | play-from-diskの拡張版。既にネゴシエーション済みのPeerConnectionに対して、動画トラックを追加/削除する方法を示します。 |
| [Insertable Streams](insertable-streams) | E2E暗号化された動画を送信し、ブラウザのInsertable Streamsで復号化するサンプル。 |
| [Save to Disk](save-to-disk) | Webカメラの映像を録画し、サーバー側でディスクに保存するサンプル。 |
| [Broadcast](broadcast) | 複数のピアに動画をブロードキャストするサンプル。配信者が一度アップロードした動画をサーバーが全ピアに転送します。 |
| [RTP Forwarder](rtp-forwarder) | RTPを使用して音声/動画ストリームを転送するサンプル。 |
| [RTP to WebRTC](rtp-to-webrtc) | Pionプロセスに送信されたRTPパケットをブラウザに配信するサンプル。 |
| [Simulcast](simulcast) | 3つのSimulcastストリームを含む1つのトラックを受信・分離し、3つの独立したトラックとして送信者に返すサンプル。 |
| [Swap Tracks](swap-tracks) | Pion Media APIの高度な使用例。サーバーが3つのメディアストリームを受け取り、動的に1つのストリームとしてユーザーにルーティングします。 |
| [RTCP Processing](rtcp-processing) | PionのRTCP APIのデモ。メディア統計情報や制御情報にアクセスできます。 |
| [Quick Switch](quick-switch) | WebRTCを使用して動画フィードを素早く切り替えるサンプル。swap-tracksと似ていますが、ユーザーが切り替えタイミングを制御し、静的ファイルを使用します。 |

#### Data Channel API

| サンプル | 説明 |
|---------|------|
| [Data Channels](data-channels) | Webブラウザとの間でDataChannelメッセージを送受信するサンプル。 |
| [Data Channels Detach](data-channels-detach) | 基盤となるDataChannel実装を直接使用してメッセージを送受信するサンプル。より慣用的な方法でData Channelを操作できます。 |
| [Data Channels Flow Control](data-channels-flow-control) | DataChannel APIを効率的に使用する方法。リモートピアがデータを受信するレートを測定し、それに応じてアプリケーションを構成できます。 |
| [ORTC](ortc) | ORTC APIを使用したDataChannel通信のサンプル。 |
| [Pion to Pion](pion-to-pion) | 2つのPionインスタンスが直接通信するサンプル。対応するWebページはありません。 |

#### その他

| サンプル | 説明 |
|---------|------|
| [Custom Logger](custom-logger) | ユーザーがログ処理をオーバーライドし、標準出力に出力する代わりにメッセージを処理する方法。対応するWebページはありません。 |
| [ICE Restart](ice-restart) | WebRTC接続がネットワーク間をローミングできることを示すサンプル。ループ内でICEを再起動し、毎回使用する新しいアドレスを表示します。 |
| [ICE Single Port](ice-single-port) | 単一のポートから複数のWebRTC接続を提供する方法。デフォルトではPionはPeerConnectionごとに新しいポートでリッスンしますが、複数の接続に単一のポートを使用するように設定できます。 |
| [ICE TCP](ice-tcp) | UDPの代わりにTCPでWebRTC接続を確立する方法。デフォルトではPionはUDPのみを使用しますが、TCPポートを使用するように設定でき、このTCPポートは多くの接続に使用できます。 |
| [ICE Proxy](ice-proxy) | TURN接続にプロキシを使用する方法。 |
| [Trickle ICE](trickle-ice) | Pion WebRTCのTrickle ICE APIのデモ。ICEの収集と接続を同時に行えるため、重要な機能です。 |
| [VNet](vnet) | Pionのネットワーク仮想化ライブラリのデモ。2つのPeerConnectionを仮想ネットワーク上で接続し、通過するデータの統計情報を表示します。 |

### 使い方

ブラウザベースのサンプルをローカルマシンで簡単に実行できます。

1. サンプルサーバーをビルドして実行:
    ```sh
    git clone https://github.com/pion/webrtc.git webrtc
    cd pion/webrtc/examples
    go run examples.go
    ```

2. [localhost](http://localhost) にアクセスしてサンプルを閲覧。`--address` フラグでサーバーのポートを変更できます:
    ```sh
    go run examples.go --address localhost:8080
    go run examples.go --address :8080            # 全インターフェースでリッスン
    ```

### WebAssembly

Pion WebRTCはWebAssembly（WASM）にコンパイルして使用できます。この場合、ライブラリはJavaScript WebRTC APIのラッパーとして動作します。これにより、サーバーサイドとブラウザサイドの両方のコードで、ほとんど変更なくGoからWebRTCを使用できます。

一部のサンプルはWebAssemblyをサポートしています。上記のサンプルサーバーを使用してWebAssemblyサンプルを実行できますが、最初にコンパイルが必要です。手順は以下の通りです:

1. サンプルがWebAssemblyをサポートしている場合、`jsfiddle` フォルダの下に `main.go` ファイルがあります。
2. この `main.go` ファイルを以下のようにビルドします:
    ```
    GOOS=js GOARCH=wasm go build -o demo.wasm
    ```
3. サンプルサーバーを起動します。ビルド方法は[使い方](#使い方)セクションを参照してください。
4. [localhost](http://localhost) にアクセスします。WebAssemblyバイナリを使用してサンプルを実行するオプションが表示されます。
