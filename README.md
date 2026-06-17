# agent-kb-template

AIエージェント（Claude Code / Codex など）と GitHub Issues を組み合わせて作業する、ナレッジベース運用の starter template。

このリポジトリを fork して自分の `agent-kb` を作り、AIとの作業をその場限りの会話で終わらせず、IssueでフローをKBでストックを蒸留する運用を始めることができる。

---

## セットアップ手順

### 1. このリポジトリを fork する

GitHub の "Use this template" ボタンから **private** リポジトリとして fork する。  
リポジトリ名は `agent-kb` を推奨。

```bash
# ローカルに clone
git clone https://github.com/{YOUR_GITHUB_LOGIN}/agent-kb.git ~/agent-kb
```

### 2. GitHub Projects を作成する

1. GitHub の自分のプロフィールページ → **Projects** → **New project** → **Board** レイアウトで作成
2. Status フィールドに以下の6択を設定する:

   | Status | 意味 |
   |---|---|
   | Inbox | 起票直後・未仕分け |
   | Todo | 仕分け済み・着手可能 |
   | In Progress | 作業中 |
   | Blocked | 停止中 |
   | Review | Agentが完了報告・人間レビュー待ち |
   | Done | 完了・close済み |

3. 以下の値を控えておく（後で使う）:
   - **Project番号**: URL の `/projects/<番号>`
   - **Project ID**: `gh api graphql -f query='query{user(login:"{YOUR_GITHUB_LOGIN}"){projectV2(number:{PROJECT_NUMBER}){id}}}'`
   - **Status フィールド ID**: `gh api graphql ...` でフィールド一覧を取得

### 3. ラベルを作成する

```bash
# type ラベル
gh label create "type:task"     --repo {YOUR_GITHUB_LOGIN}/agent-kb --color 5319e7
gh label create "type:decision" --repo {YOUR_GITHUB_LOGIN}/agent-kb --color 0052cc
gh label create "type:research" --repo {YOUR_GITHUB_LOGIN}/agent-kb --color 0e8a16
gh label create "type:bug"      --repo {YOUR_GITHUB_LOGIN}/agent-kb --color d73a4a
gh label create "type:chore"    --repo {YOUR_GITHUB_LOGIN}/agent-kb --color e4e669

# priority ラベル
gh label create "priority:high"   --repo {YOUR_GITHUB_LOGIN}/agent-kb --color b60205
gh label create "priority:medium" --repo {YOUR_GITHUB_LOGIN}/agent-kb --color fbca04
gh label create "priority:low"    --repo {YOUR_GITHUB_LOGIN}/agent-kb --color c5def5
```

### 4. プレースホルダーを書き換える

以下のファイルの `{...}` を自分の値に置き換える。

| ファイル | プレースホルダー |
|---|---|
| `AGENT_GUIDE.md` | `{YOUR_NAME}` / `{YOUR_COMPANY}` / `{YOUR_EMAIL}` |
| `patterns/issue_based_agent_workflow.md` | `{YOUR_GITHUB_LOGIN}` / `{PROJECT_NUMBER}` / `{PROJECT_ID}` / `{STATUS_FIELD_ID}` |

### 5. スラッシュコマンドを配置する

`.claude/commands/` 内の3ファイルを `~/.claude/commands/` にコピーし、同じプレースホルダーを置き換える。

```bash
cp ~/agent-kb/.claude/commands/*.md ~/.claude/commands/
# その後 ~/.claude/commands/ 内の {YOUR_GITHUB_LOGIN} 等を書き換える
```

### 6. AI の設定ファイルに読み込み指示を追加する

`~/.claude/CLAUDE.md`（Claude Code）または `~/AGENTS.md`（Codex）に以下を追記する。

```
## ナレッジベース

タスク開始時に必ず読み込む:
- ~/agent-kb/AGENT_GUIDE.md
- ~/agent-kb/INDEX.md
```

---

## ディレクトリ構成

```
agent-kb/
  AGENT_GUIDE.md              ← AIが毎回読む共通ルール
  INDEX.md                    ← 全エントリの索引
  patterns/
    issue_based_agent_workflow.md   ← Issue起点のAgent作業ルール
    kb_document_design.md           ← KB文書の設計・更新ルール
    information_management.md       ← フロー/ストック分離パターン
  initiatives/
    _registry.md              ← initiative一覧の正本
  tools/                      ← ツール・SaaS・APIの知識
  company/                    ← 組織・事業・判断基準
  reference/                  ← 外部情報・人物プロファイル等
  archive/                    ← 完了済み・廃止済みの文書
  .claude/commands/
    start-issue.md            ← /start-issue スラッシュコマンド
    close-issue.md            ← /close-issue スラッシュコマンド
    file-issue.md             ← /file-issue スラッシュコマンド
```

---

## 使い方

セットアップ後は、Claude Code（または Codex）で以下のコマンドが使えるようになる。

| コマンド | 用途 |
|---|---|
| `/file-issue <タイトル>` | Issueを対話的に整理して起票 |
| `/start-issue <番号>` | Issue着手前プロトコル（KB読込・Status更新） |
| `/close-issue <番号>` | 完了処理（KB更新・コメント・close） |
