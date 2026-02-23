# WSNet2 Getting Started

WSNet2をローカル環境で起動し、デモを実行するための手順です。

## 前提条件

以下のソフトウェアがインストールされている必要があります：

- **Docker & Docker Compose** - サーバー群の起動用
- **Node.js & npm** - ダッシュボードのフロントエンド用
- **Unity** (オプション) - クライアントアプリの実行用

## クイックスタート

### 1. フロントエンドのビルド

まず、ダッシュボード用のフロントエンドをビルドします：

```bash
# フロントエンドディレクトリに移動
cd wsnet2-dashboard/frontend

# 依存関係をインストール
npm install

# ビルド実行
npm run build

# distフォルダが作成されることを確認
ls -la dist/
```

### 2. サーバー群の起動

すべてのWSNet2サーバーをDockerで起動します：

```bash
# プロジェクトルートに戻る
cd ../..

# wsnet2-dashboardディレクトリに移動
cd wsnet2-dashboard

# すべてのサービスを起動（デタッチモード）
docker compose -f compose.yaml -f ../server/compose.yaml up -d
```

### 3. 起動確認

サービスが正常に起動したことを確認します：

```bash
# コンテナの状態を確認
docker compose ps

# ログを確認（オプション）
docker compose logs -f
```

### 4. ダッシュボードにアクセス

ブラウザで以下のURLにアクセスします：

```
http://localhost:8081
```

## サービス一覧

起動後、以下のサービスが利用可能になります：

| サービス | URL | 説明 |
|---------|-----|------|
| ダッシュボード | http://localhost:8081 | Web管理画面 |
| Lobbyサーバー | http://localhost:8080 | API エンドポイント |
| Gameサーバー | ws://localhost:8000 | WebSocket接続 |
| Hubサーバー | ws://localhost:8001 | 観戦用WebSocket |
| MySQL | localhost:3306 | データベース |

## Unityクライアントの実行

### 1. Unityでプロジェクトを開く

```bash
# UnityでwsNet2-unityディレクトリを開く
# Unity Hub → Add → wsnet2-unity を選択
```

### 2. サンプルシーンの実行

1. Unityで `Assets/Sample/Title.unity` シーンを開く
2. Playボタンを押してシーンを実行
3. 設定画面で以下を入力：
   - **Lobby**: `http://localhost:8080`
   - **AppID**: `testapp`
   - **Key**: `testapppkey`
   - **UserID**: 任意のユーザーID（例：`user001`）

### 3. ゲームプレイ

- **CPU対戦**: オフラインでCPUと対戦
- **部屋作成**: オンラインで他のプレイヤーを待機
- **ランダム入室**: 既存の部屋に参加
- **ランダム観戦**: 進行中のゲームを観戦

## .NETサンプルの実行

### Bot との対戦

```bash
# .NETサンプルディレクトリに移動
cd wsnet2-dotnet/WSNet2.Sample

# Botを起動（ランダム入室）
dotnet run -- -b

# マスタークライアント起動（部屋作成＋Bot参加）
dotnet run -- -m -b
```

### パラメータ説明

- `-s URL`: サーバーURL（デフォルト: http://localhost:8080）
- `-m [数]`: マスタークライアントの数
- `-b [数]`: Botの数

## トラブルシューティング

### よくある問題と解決方法

#### 1. 403 Forbidden (localhost:8081)

**原因**: フロントエンドがビルドされていない

**解決方法**:
```bash
cd wsnet2-dashboard/frontend
npm install
npm run build
```

#### 2. Go バージョンエラー

**エラー**: `requires go >= 1.24.0`

**解決方法**: Dockerfileは既にGo 1.24を使用しているため、通常は問題ありません。

#### 3. ポート競合

**エラー**: `port already in use`

**解決方法**:
```bash
# 使用中のポートを確認
lsof -i :8080
lsof -i :8081

# 競合するプロセスを停止するか、別のポートを使用
```

#### 4. データベース接続エラー

**解決方法**:
```bash
# MySQLコンテナの状態確認
docker compose ps wsnet2-db

# ログ確認
docker compose logs wsnet2-db
```

### コンテナの停止・再起動

```bash
# 全サービス停止
docker compose -f compose.yaml -f ../server/compose.yaml down

# 特定のサービスのみ再起動
docker compose restart wsnet2-lobby

# データベースも含めて完全にリセット
docker compose -f compose.yaml -f ../server/compose.yaml down -v
```

## 開発・デバッグ

### ログの確認

```bash
# 全サービスのログを表示
docker compose logs -f

# 特定のサービスのログのみ
docker compose logs -f wsnet2-lobby
docker compose logs -f wsnet2-game
docker compose logs -f wsnet2-hub
```

### データベースの確認

```bash
# MySQLに接続
docker compose exec wsnet2-db mysql -u wsnet2 -pwsnet2pass wsnet2

# テーブル確認
SHOW TABLES;
SELECT * FROM room;
SELECT * FROM app;
```

### Bot負荷テスト

```bash
# サーバーのbinディレクトリからBotツールを実行
cd server/bin

# 負荷テスト実行（10部屋、各2プレイヤー、5観戦者）
./wsnet2-bot load -c 10 -p 2 -w 5

# シナリオテスト実行
./wsnet2-bot scenario
```

## 次のステップ

1. **コード解析**: [`CODE_ANALYSIS.md`](CODE_ANALYSIS.md) でアーキテクチャを理解
2. **ドキュメント**: [`_doc/`](_doc/) で詳細な仕様を確認
3. **カスタマイズ**: 設定ファイル（`server/docker.toml`）を調整
4. **開発**: 独自のゲームロジックを実装

## 参考リンク

- [プロジェクトREADME](README.md)
- [サーバー設定](server/README.md)
- [ダッシュボード](wsnet2-dashboard/README-ja.md)
- [Unity クライアント](_doc/csharp_client.md)
