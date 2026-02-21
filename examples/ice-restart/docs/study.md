# ice-restart

このサンプルは **ICE Restart（ICE再起動）** 機能を示すデモです。

## ICE Restartとは？

WebRTC接続中にネットワーク状況が変化した場合（WiFi切り替え、IPアドレス変更など）、**接続を切断せずに新しいICE候補を再収集して接続を維持・回復** する機能です。

## 構成

| ファイル | 役割 |
|----------|------|
| `main.go` | Go側サーバー（Answer側、DataChannel送信） |
| `index.html` | ブラウザ側（Offer側、ICE Restart実行） |

## アーキテクチャ

```
┌─────────────────┐                    ┌─────────────────┐
│    Browser      │                    │    Go Server    │
│   (Offer側)     │                    │   (Answer側)    │
├─────────────────┤                    ├─────────────────┤
│ RTCPeerConnection│◄──── SDP交換 ────►│ PeerConnection  │
│ DataChannel     │◄── DataChannel ───►│ OnDataChannel   │
└─────────────────┘                    └─────────────────┘
        │                                      │
        │  [ICE Restart] ボタン押下            │
        │                                      │
        ▼                                      ▼
  iceRestart: true                    SetRemoteDescription
  で Offer 再作成                      + CreateAnswer
```

## 処理フロー

1. ブラウザがOffer作成・送信
2. Go側がAnswer返却
3. DataChannelで3秒ごとにメッセージ送受信
4. [ICE Restart]ボタン押下 → `iceRestart: true` でOffer再作成
5. 新しいICE候補が収集され、接続が更新される

## コード詳細

### ブラウザ側 - ICE Restartの実行（index.html）

```javascript
let pc = new RTCPeerConnection({
  iceServers: [{
    urls: 'stun:stun.l.google.com:19302'
  }]
})

let dc = pc.createDataChannel('data')

// ICE Restartを実行する関数
window.doSignaling = iceRestart => {
  // iceRestart: true を指定してOfferを作成
  pc.createOffer({iceRestart})
    .then(offer => {
      pc.setLocalDescription(offer)
      return fetch(`/doSignaling`, {
        method: 'post',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify(offer)
      })
    })
    .then(res => res.json())
    .then(res => pc.setRemoteDescription(res))
}

// 初回接続（iceRestart: false）
window.doSignaling(false)
```

### ブラウザ側 - 選択されたICEペアの表示

```javascript
dc.onopen = () => {
  setInterval(function() {
    // 現在選択されているICE候補ペアを取得
    let selectedPair = pc.sctp.transport.iceTransport.getSelectedCandidatePair()
    // Local/Remoteの候補を表示
    console.log('Local:', selectedPair.local.candidate)
    console.log('Remote:', selectedPair.remote.candidate)
  }, 3000);
}
```

ICE Restart後、uFrag/uPwd/ポートが変化することを確認できる。

### Go側 - 同一PeerConnectionの再利用（main.go）

```go
var peerConnection *webrtc.PeerConnection

func doSignaling(res http.ResponseWriter, req *http.Request) {
    // 初回のみPeerConnectionを作成、以降は再利用
    if peerConnection == nil {
        peerConnection, _ = webrtc.NewPeerConnection(webrtc.Configuration{})

        peerConnection.OnICEConnectionStateChange(func(state webrtc.ICEConnectionState) {
            fmt.Printf("ICE Connection State has changed: %s\n", state.String())
        })

        peerConnection.OnDataChannel(func(d *webrtc.DataChannel) {
            d.OnOpen(func() {
                // 3秒ごとに現在時刻を送信
                for range time.Tick(time.Second * 3) {
                    d.SendText(time.Now().String())
                }
            })
        })
    }

    // Offerを受け取ってAnswerを返す（ICE Restartでも同じ処理）
    var offer webrtc.SessionDescription
    json.NewDecoder(req.Body).Decode(&offer)
    peerConnection.SetRemoteDescription(offer)

    gatherComplete := webrtc.GatheringCompletePromise(peerConnection)
    answer, _ := peerConnection.CreateAnswer(nil)
    peerConnection.SetLocalDescription(answer)
    <-gatherComplete

    response, _ := json.Marshal(*peerConnection.LocalDescription())
    res.Write(response)
}
```

## 学べること

### 1. ICE Restartの実行方法

```javascript
// ブラウザ側: iceRestart: true を指定するだけ
pc.createOffer({iceRestart: true})
```

### 2. ICE Restartで変化するもの

| 項目 | 変化 |
|------|------|
| uFrag/uPwd | 新しい値に更新 |
| ポート番号 | 新しいポートが使用される可能性 |
| ICE候補 | 新しい候補が収集される |

### 3. 接続状態の遷移

ICE Restart時の典型的な状態遷移:
```
connected → checking → connected
```

### 4. 同一PeerConnectionの再利用

Go側でPeerConnectionを再作成せずに再利用している点に注目。

## 同一PeerConnectionを再利用するメリット

### 1. 既存のメディア/データストリームが維持される

```
PeerConnection再作成の場合:
  → DataChannel切断 → 再作成・再接続が必要
  → MediaTrack切断 → 再ネゴシエーションが必要

ICE Restart（再利用）の場合:
  → DataChannel維持 ✓
  → MediaTrack維持 ✓
```

**ユーザー体験**: 通話やデータ送信が途切れない

### 2. DTLS/SRTPセッションの維持

| 項目 | 再作成 | ICE Restart |
|------|--------|-------------|
| DTLSハンドシェイク | 再実行（重い） | 不要 |
| SRTP鍵 | 再生成 | 維持 |
| 暗号化セッション | リセット | 継続 |

**パフォーマンス**: 暗号化の再ネゴシエーションコストを回避

### 3. シグナリングが軽量

```javascript
// ICE Restart: Offer/Answerのみ
pc.createOffer({iceRestart: true})

// 再作成: 全てやり直し
// - PeerConnection作成
// - DataChannel/Track追加
// - Offer/Answer
// - ICE候補交換
```

### 4. アプリケーション状態の維持

```go
// 再作成の場合: イベントハンドラを再設定する必要がある
peerConnection.OnTrack(...)
peerConnection.OnDataChannel(...)
peerConnection.OnICEConnectionStateChange(...)

// ICE Restart: 既存のハンドラがそのまま動作
```

### 5. 接続ダウンタイムの最小化

```
再作成:    [切断]----[セットアップ]----[接続]  (数秒のダウンタイム)
ICE Restart: [接続維持]--[新候補探索]--[切替]  (ほぼシームレス)
```

### 比較まとめ

| 観点 | 再作成 | ICE Restart（再利用） |
|------|--------|----------------------|
| DataChannel/Track | 再構築 | 維持 |
| 暗号化セッション | 再ネゴ | 維持 |
| コード複雑度 | 高い | 低い |
| 切り替え時間 | 長い | 短い |
| ユーザー体験 | 中断あり | シームレス |

## ユースケース

- **モバイル端末**: WiFiとLTE間の切り替え時
- **ノートPC**: ネットワーク変更時
- **長時間接続**: 定期的なICE Restartで接続を健全に保つ
- **NAT/ファイアウォール変更**: IPアドレスが変わった場合

## 実行方法

```bash
cd examples/ice-restart
go run *.go
```

ブラウザで http://localhost:8080 を開く。

## UI要素

- **ICE Restart** ボタン: `iceRestart: true` でOfferを再作成
- **ICE Connection States**: 接続状態の遷移履歴
- **ICE Selected Pairs**: 3秒ごとに選択されたICE候補ペアを表示
- **Inbound DataChannel Messages**: Go側から送信された現在時刻

## 結論

ネットワーク変更時は、PeerConnection再作成ではなくICE Restartを使うべき。接続を維持しながらネットワークパスを更新できる。
