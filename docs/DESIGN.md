# tui-todo 設計方針

このドキュメントは、tui-todo の開発を引き継ぐ AI エージェント (Codex 等) や人間の開発者が、コードベースの「形」と「守るべき制約」を短時間で把握するためのものです。

## 1. プロダクト概要

WSL / Linux / macOS のターミナルで動く TODO アプリ。中心機能は **サブタスク (チェックリスト) を持ったタスク** と、その雛形となる **テンプレート** の管理。テンプレートからワンタッチでタスクを生成できる点が他の TUI TODO との差別化点。

## 2. 設計原則

これらは合意済みの大方針。逸脱する場合は事前に確認すること。

1. **環境依存を最小化する**
   - Go 製。`CGO_ENABLED=0` で静的バイナリ 1 個に固める。
   - 外部 CLI (`sqlite3`, `git`, `xclip` 等) に依存しない。
   - データは JSON ファイル。スキーマ移行は将来必要になったら検討。
2. **依存ライブラリは最小限**
   - 採用済: `bubbletea`, `bubbles`, `lipgloss`, `google/uuid` のみ。
   - 追加する場合はメンテ状況とライセンス (MIT/BSD/Apache-2 のみ) を確認。
3. **ロジックと UI を分離**
   - `internal/model`, `internal/storage`, `internal/service` は UI を一切知らない。
   - `internal/ui` だけが Bubble Tea に依存する。
   - これによりロジックは通常の Go テストで検証できる。
4. **過剰な抽象化を避ける**
   - インターフェースは「2 個目の実装が現れた時点で」切る。
   - 設定ファイル・プラグイン機構・DI コンテナ等はまだ要らない。
5. **破壊的操作は確認を伴う設計に**
   - 削除には今後 `?` などの確認プロンプトを挟む余地を残す（現状は即時削除、TODO）。

## 3. ディレクトリ構成

```text
tui-todo/
├── main.go
├── go.mod / go.sum
├── Makefile
├── README.md
├── .gitignore
├── docs/
│   ├── DESIGN.md
│   └── CODEX_PROMPT.md
├── internal/
│   ├── model/
│   │   └── model.go
│   ├── storage/
│   │   └── storage.go
│   ├── service/
│   │   ├── instantiate.go
│   │   └── instantiate_test.go
│   └── ui/
│       ├── app.go
│       └── styles.go
└── scripts/
    ├── build.sh
    ├── install.sh
    ├── run.sh
    └── uninstall.sh
```

## 4. データモデル

`internal/model/model.go` の構造体が単一の真実 (single source of truth)。

```go
type Subtask struct {
    Title string
    Done  bool
}

type Task struct {
    ID         string
    TemplateID string
    Title      string
    Done       bool
    Subtasks   []Subtask
    CreatedAt  time.Time
}

type Template struct {
    ID       string
    Name     string
    Subtasks []Subtask
}
```

### スキーマ変更ルール

- 既存フィールドは削らない（古い JSON 互換性のため）。
- 新フィールドは `omitempty` を付ける。
- 破壊的変更が必要になったらマイグレーション関数を `internal/storage` に追加する。

## 5. 永続化

- 場所: `os.UserConfigDir()/tui-todo/` (Linux/WSL: `~/.config/tui-todo/`)
- 環境変数 `TUI_TODO_DIR` でオーバーライド可能（テストや配布で使用）。
- ファイル: `tasks.json`, `templates.json` (`encoding/json` の `MarshalIndent`)。
- 書き込みは `tmp` ファイルを書いてから `Rename` する atomic write。

## 6. UI とモード遷移

`internal/ui/app.go` に単一の `Model` がある。

- `viewTasks` → `t` → `viewTemplates`
- `viewTemplates` → `e`/`Enter` → `viewTemplateEdit`
- `esc`/`t` で戻る

加えて、任意のモードから「入力プロンプト (`promptKind`)」が一時的に開く。プロンプト表示中はキー入力はすべて `textinput` に渡す。

## 7. 実装済みの機能

- タスク: 追加 / 完了トグル / 改名 / 削除 / サブタスク追加
- サブタスク: 完了トグル / 改名 / 削除 / 展開折りたたみ / 進捗表示 (`(2/5)`)
- テンプレート: 追加 / 改名 / 削除 / 編集
- テンプレート編集: サブタスクの追加 / 改名 / 削除 / 並び替え (`J`/`K`)
- テンプレート → タスク生成 (`N` キー、サブタスクは未完了状態でコピー)
- atomic な JSON 永続化

## 8. 未実装で歓迎する拡張

1. 削除確認プロンプト
2. 検索 / フィルタ
3. タグまたはカテゴリ
4. 期限と並び替え
5. アンドゥ
6. JSON 以外のエクスポート
7. 設定ファイル

## 9. テスト方針

- ロジック層 (`internal/service`, `internal/storage`) は通常の Go テストで覆う。
- UI 層 (`internal/ui`) は必要になったら `tea.Msg` を直接流し込むステート検証テストを追加する。
- カバレッジ目標は設けない。バグ修正や機能追加には回帰テスト 1 本を推奨。
