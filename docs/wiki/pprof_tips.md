# pprof Tips (wsnet2)

## 前提

- `server/docker.toml` で `Lobby.pprof_port = 3000`
- `server/compose.yaml` で `lobby` は `3080:3000` を公開
- そのため、ホスト側からは `http://localhost:3080` で参照する

## まず確認する

```bash
curl http://localhost:3080/debug/pprof/
```

ブラウザで一覧を見る場合:

```text
http://localhost:3080/debug/pprof/
```

## よく使うコマンド

CPU (30秒サンプリング):

```bash
go tool pprof -http=:0 "http://localhost:3080/debug/pprof/profile?seconds=30"
```

Heap:

```bash
go tool pprof -http=:0 http://localhost:3080/debug/pprof/heap
```

Goroutine:

```bash
go tool pprof -http=:0 http://localhost:3080/debug/pprof/goroutine
```

軽くテキストで確認:

```bash
curl http://localhost:3080/debug/pprof/goroutine?debug=1 | head -n 40
```

## 見るポイント

- CPUが高い: `profile` の `Top` と `Flame Graph` で重い関数を特定
- メモリが増える: `heap` の `Top` で割当が多い箇所を確認
- ハング/遅延: `goroutine` で待ち続けているスタックを確認

## 注意

- `pprof` は内部情報を返すため、公開環境ではアクセス制限する
- 調査不要な環境は `pprof_port = 0` で無効化可能

## Game / Hub の参照先

compose 定義では次のポートで参照できる:

- Game: `localhost:3000`（`3000:3000`）
- Hub: `localhost:3010`（`3010:3000`）

例:

```bash
# Game の CPU
go tool pprof -http=:0 "http://localhost:3000/debug/pprof/profile?seconds=30"

# Hub の Heap
go tool pprof -http=:0 http://localhost:3010/debug/pprof/heap
```
