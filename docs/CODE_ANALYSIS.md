# WSNet2 コード解析レポート

## 概要

WSNet2は、WebSocketをベースとしたモバイルオンラインゲーム向けのリアルタイム通信システムです。分散アーキテクチャを採用し、高いスケーラビリティと信頼性を提供します。

## アーキテクチャ概要

### システム構成

WSNet2は以下の3つの主要コンポーネントで構成されています：

1. **Lobbyサーバー** (`server/cmd/wsnet2-lobby/`)
   - 部屋の作成、検索、参加リクエストを処理
   - ゲーム/ハブサーバーとの仲介役
   - HTTP APIとgRPCクライアント機能を提供

2. **Gameサーバー** (`server/cmd/wsnet2-game/`)
   - 実際のゲームルームを提供
   - WebSocket接続でプレイヤーとの通信
   - リアルタイムメッセージング機能

3. **Hubサーバー** (`server/cmd/wsnet2-hub/`)
   - 観戦機能を提供
   - 多数の観戦者を効率的にサポート
   - GameサーバーからのブロードキャストをHubが中継

### 通信プロトコル

- **外部API**: HTTP + MessagePack
- **内部通信**: gRPC
- **クライアント接続**: WebSocket

## サーバーサイド実装詳細

### 1. Lobbyサーバー

**主要ファイル:**
- `server/lobby/service/api.go` - HTTP APIエンドポイント
- `server/lobby/room.go` - ルーム管理ロジック

**主要機能:**
- 部屋作成 (`POST /rooms`)
- 部屋参加 (`POST /rooms/join/id/{roomId}`)
- ランダム参加 (`POST /rooms/join/random`)
- 部屋検索 (`POST /rooms/search`)
- 観戦開始 (`POST /rooms/watch/id/{roomId}`)

**実装例 - 部屋作成:**
```go
func (rs *RoomService) Create(ctx context.Context, appId string, roomOption *pb.RoomOption, clientInfo *pb.ClientInfo, macKey string) (*pb.JoinedRoomRes, error) {
    // 1. アプリケーション確認
    if _, found := rs.apps[appId]; !found {
        return nil, xerrors.Errorf("Unknown appId: %v", appId)
    }

    // 2. 利用可能なGameサーバーを選択
    game, err := rs.gameCache.Rand()
    if err != nil {
        return nil, xerrors.Errorf("get game server: %w", err)
    }

    // 3. GameサーバーにgRPCで部屋作成要求
    client, err := rs.newGameClient(game.Hostname, game.GRPCPort)
    if err != nil {
        return nil, xerrors.Errorf("newGameClient: %w", err)
    }

    req := &pb.CreateRoomReq{
        AppId:      appId,
        RoomOption: roomOption,
        MasterInfo: clientInfo,
        MacKey:     macKey,
    }

    res, err := client.Create(ctx, req)
    // ...エラー処理とレスポンス返却
}
```

### 2. Gameサーバー

**主要ファイル:**
- `server/game/service/grpc.go` - gRPCサービス実装
- `server/game/service/websocket.go` - WebSocket処理
- `server/game/room.go` - ルーム管理
- `server/game/client.go` - クライアント管理

**主要機能:**
- ルーム作成・管理
- プレイヤー接続管理
- メッセージブロードキャスト
- RPC処理

**実装例 - WebSocket接続処理:**
```go
func (s *WSHandler) HandleRoom(w http.ResponseWriter, r *http.Request) {
    roomId := r.PathValue("id")
    appId := r.Header.Get("Wsnet2-App")
    clientId := r.Header.Get("Wsnet2-User")
    
    // 1. クライアント認証
    cli, err := repo.GetClient(roomId, clientId)
    if err != nil {
        http.Error(w, "Not Found", http.StatusNotFound)
        return
    }

    // 2. WebSocket升级
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        logger.Errorf("websocket: upgrade: %+v", err)
        return
    }

    // 3. Peer作成してメッセージ処理開始
    peer, err := game.NewPeer(ctx, cli, conn, lastEvSeq)
    if err != nil {
        logger.Warnf("websocket: NewPeer: %+v", err)
        return
    }
    <-peer.Done()
}
```

**ルーム管理システム:**
```go
type Room struct {
    *pb.RoomInfo
    repo *Repository
    
    publicProps  binary.Dict
    privateProps binary.Dict
    
    msgCh    chan Msg
    done     chan struct{}
    
    muClients   sync.RWMutex
    players     map[ClientID]*Client
    master      *Client
    masterOrder []ClientID
    watchers    map[ClientID]*Client
    
    lastMsg binary.Dict
    logger log.Logger
}
```

### 3. Hubサーバー

**主要ファイル:**
- `server/hub/repository.go` - Hub管理
- `server/hub/service/websocket.go` - 観戦者向けWebSocket

**機能:**
- 観戦者の接続管理
- GameサーバーからのメッセージをHubが中継
- 大規模観戦のスケーラビリティ確保

## クライアントサイド実装

### 1. Unity クライアント (`wsnet2-unity/`)

**主要コンポーネント:**
- `WSNet2Client.cs` - メインクライアントクラス
- `Room.cs` - ルーム管理
- `Player.cs` - プレイヤー情報

**実装例 - 部屋作成:**
```csharp
public void Create(
    RoomOption roomOption,
    IDictionary<string, object> clientProps,
    Action<Room> onSuccess,
    Action<Exception> onFailed,
    IWSNet2Logger<WSNet2LogPayload> roomLogger)
{
    var authData = this.authData;
    var param = new CreateParam()
    {
        roomOption = roomOption,
        clientInfo = new ClientInfo(userId, clientProps),
        encryptedMACKey = authData.EncryptedMACKey,
    };
    var content = MessagePackSerializer.Serialize(param);

    Task.Run(() => connectToRoom("/rooms", content, authData, onSuccess, onFailed, roomLogger));
}
```

**RPC システム:**
```csharp
// RPC登録
public int RegisterRPC<T>(Action<string, T> rpc) where T : IWSNet2Serializable
{
    var id = registerRPC(rpc, (senderId, reader) => 
        rpc(senderId, WSNet2Serializer.Read<T>(reader)));
    registerRPCMethodName(id, rpc);
    return id;
}

// RPC呼び出し
public int RPC<T>(Action<string, T> rpc, T param, params string[] targets)
    where T : IWSNet2Serializable
{
    var id = getRpcId(rpc);
    var payload = WSNet2Serializer.Write(param);
    return sendRPC(id, payload, targets);
}
```

### 2. .NET クライアント (`wsnet2-dotnet/`)

**主要コンポーネント:**
- `WSNet2.Client/` - クライアントライブラリ
- `WSNet2.Sample/` - サンプル実装
  - `MasterClient.cs` - ゲームマスター
  - `BotClient.cs` - Bot実装

**サンプルゲーム実装:**
```csharp
class MasterClient : IMasterClient
{
    // ゲームロジック駆動
    void GameUpdate()
    {
        var prevStateCode = state.Code;
        simulator.Update(state, events, timer.NowTick);

        // 定期的にステート同期
        var now = timer.NowTick;
        if (now - lastSync >= SyncInterval)
        {
            rpc?.SyncServerTick(timer.NowTick);
            rpc?.SyncGameState(state);
            lastSync = now;
        }

        // ステート変更をパブリックプロパティに反映
        if (room != null && prevStateCode != state.Code)
        {
            room.ChangeRoomProperty(publicProps: new Dictionary<string, object>
            {
                {WSNet2Helper.PubKey.State, state.Code.ToString()}
            });
        }
    }
}
```

## データベーススキーマ

**主要テーブル:**
```sql
-- server/sql/10-schema.sql より
CREATE TABLE `room` (
  `id` char(26) NOT NULL,
  `app_id` varchar(255) NOT NULL,
  `host_id` int(10) unsigned NOT NULL,
  `visible` tinyint(1) NOT NULL DEFAULT '1',
  `joinable` tinyint(1) NOT NULL DEFAULT '1',
  `watchable` tinyint(1) NOT NULL DEFAULT '1',
  `number` int(10) unsigned DEFAULT NULL,
  `search_group` int(10) unsigned NOT NULL DEFAULT '0',
  `max_players` int(10) unsigned NOT NULL,
  `players` int(10) unsigned NOT NULL DEFAULT '0',
  `watchers` int(10) unsigned NOT NULL DEFAULT '0',
  `public_props` longblob,
  `private_props` longblob,
  `created` datetime(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (`id`),
  UNIQUE KEY `number` (`app_id`,`number`),
  KEY `idx_search` (`app_id`,`search_group`,`joinable`,`visible`,`players`,`max_players`)
);
```

## プロトコルとメッセージフォーマット

### HTTP API

**部屋作成例:**
```http
POST /rooms
Content-Type: application/x-msgpack
Wsnet2-App: testapp
Wsnet2-User: user001

[MessagePack encoded CreateParam]
```

**レスポンス:**
```json
{
  "type": "ok",
  "roomInfo": {
    "id": "room_id",
    "appId": "testapp",
    "visible": true,
    "joinable": true,
    "maxPlayers": 4
  },
  "players": [...],
  "authKey": "auth_token",
  "masterId": "user001"
}
```

### WebSocket プロトコル

**接続:**
```
WebSocket: ws://gameserver:8000/room/{roomId}
Headers:
  Wsnet2-App: testapp
  Wsnet2-User: user001
  Authorization: Bearer {authKey}
  Wsnet2-LastEventSeq: 0
```

**メッセージフォーマット:**
- バイナリメッセージを使用
- MessagePackでシリアライズ
- イベントベースの通信

## 設定とデプロイ

### Docker構成

**docker-compose.yaml:**
```yaml
version: '3.8'
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wsnet2
      MYSQL_USER: wsnet2
      MYSQL_PASSWORD: wsnet2pass

  lobby:
    build: .
    command: ["./wsnet2-lobby", "docker.toml"]
    ports:
      - "8080:8080"

  game:
    build: .
    command: ["./wsnet2-game", "docker.toml"]
    ports:
      - "8000:8000"

  hub:
    build: .
    command: ["./wsnet2-hub", "docker.toml"]
    ports:
      - "8001:8001"
```

### 設定ファイル例

**docker.toml:**
```toml
[db]
host = "db"
port = 3306
dbname = "wsnet2"
user = "wsnet2"
password = "wsnet2pass"

[lobby]
hostname = "lobby"
port = 8080
loglevel = 3

[game]
hostname = "${WSNET2_GAME_PUBLICNAME:game}"
grpc_port = 19000
websocket_port = 8000
max_rooms = 1000

[hub]
hostname = "hub" 
grpc_port = 19001
websocket_port = 8001
```

## ツールとユーティリティ

### Bot/負荷テストツール

**wsnet2-bot:**
```bash
# 負荷テスト実行
./wsnet2-bot load -c 10 -p 2 -w 5

# シナリオテスト
./wsnet2-bot scenario
```

**wsnet2-tool:**
```bash
# プレイヤーをキック
./wsnet2-tool kick {clientId}

# ルーム情報取得
./wsnet2-tool room-info {roomId}
```

## パフォーマンス特性

### スケーラビリティ

1. **水平スケーリング**
   - Gameサーバーを複数台構成可能
   - Hubサーバーで観戦者の負荷分散
   - Lobbyサーバーがロードバランサー的役割

2. **接続管理**
   - WebSocketベースの持続接続
   - 自動再接続機能
   - ハートビート機能

3. **メッセージング**
   - バイナリプロトコルで効率的通信
   - RPC系統でタイプセーフな通信
   - ブロードキャスト最適化

### 信頼性機能

1. **認証システム**
   - MACベースの認証
   - クライアント認証トークン
   - セッション管理

2. **エラー処理**
   - タイムアウト処理
   - 切断時の自動クリーンアップ
   - エラー状況の詳細ログ

3. **データ整合性**
   - データベーストランザクション
   - 楽観的ロック
   - イベント順序保証

## 開発・運用観点

### 監視・ログ

**ログ設定:**
```go
// structured logging with zap
logger := log.GetLoggerWith(
    log.KeyHandler, "grpc:Create",
    log.KeyApp, appId,
    log.KeyClient, clientId,
    log.KeyRoom, roomId,
    log.KeyRequestedAt, timestamp,
)
```

**メトリクス:**
- 接続数監視
- レスポンス時間計測
- エラー率追跡

### テスト戦略

1. **単体テスト**
   - Go言語: `*_test.go`
   - C#: NUnit/MSTest

2. **統合テスト**
   - end-to-endシナリオ
   - 負荷テスト

3. **Bot型テスト**
   - 自動化されたプレイヤーBots
   - 大規模同時接続テスト

## まとめ

WSNet2は、以下の特徴を持つ堅牢なリアルタイム通信システムです：

1. **モジュラー設計**: Lobby/Game/Hub の役割分離
2. **スケーラブル**: 水平分散とHub型観戦システム
3. **信頼性**: 自動再接続、エラーハンドリング
4. **開発体験**: Unity/.NET 両対応、豊富なサンプル
5. **運用性**: Docker化、監視機能、管理ツール

リアルタイムゲームやライブアプリケーションの基盤として、高いパフォーマンスと運用性を提供する設計となっています。
