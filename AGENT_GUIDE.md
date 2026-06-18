# KB Agent Guide

この文書は、あなたの作業を支援するAIエージェントが毎回読む共通ルールである。

目的は、AIとの作業をその場限りの会話で終わらせず、Issueでフローを管理し、KBへストックを蒸留して、次回以降のAIが文脈を引き継げる状態を作ること。

---

## 1. 基本原則

- Issueはフローの正本。未解決事項、確認事項、進行状況、完了報告を管理する。
- KBはストックの正本。現在有効な知識、設計判断、運用ルール、背景を管理する。
- KBは作業ログの置き場ではない。古い情報は現行正本から削除し、必要なら別文書へ退避する。
- 未確定事項、確認事項、次アクションはKB本文に抱えず、Issue化する。
- 追記で膨らませるのではなく、現在の理解に合わせて文書全体を再構成する。

詳細:

- KB文書の設計・作成・更新ルール: `patterns/kb_document_design.md`
- Issue起点のAgent作業ルール: `patterns/issue_based_agent_workflow.md`
- API・外部サービス操作の横断基本動作（記録値で楽観実行→失敗時だけ確認）: `patterns/api_usage_basics.md`
- GitHub Projects / labels / `gh` 操作・安定キー: `tools/github_projects.md`

---

## 2. ユーザー情報

<!-- セットアップ時に自分の情報に書き換える -->
- 名前: {YOUR_NAME}
- 会社: {YOUR_COMPANY}
- メール: {YOUR_EMAIL}

---

## 3. KBの正本

KBの作業マスター:

```text
~/agent-kb/
```

主な構造:

```text
agent-kb/
  INDEX.md
  AGENT_GUIDE.md
  company/
  initiatives/
  tools/
  patterns/
  reference/
  archive/
```

重要な追記・更新は必ず `~/agent-kb/` 側で行い、commit + push まで完了させる。

---

## 4. タスク開始時

すべてのタスク開始時に次を読む。

1. `~/agent-kb/AGENT_GUIDE.md`
2. `~/agent-kb/INDEX.md`

`INDEX.md` を見て、タスクに関係するKBエントリがあれば作業前に読む。

会社、プロジェクト、ツール、過去の判断、Issue運用に関わる場合は、関連KBを読まずに進めない。

---

## 5. KBカテゴリ

| ディレクトリ | 役割 |
|---|---|
| `company/` | 組織・事業・人・判断基準 |
| `initiatives/` | 進行中の取り組み・プロジェクト文脈 |
| `tools/` | ツール・SaaS・API・ローカル環境 |
| `patterns/` | 複数領域で再利用できる進め方・判断パターン |
| `reference/` | 正本ではないが参照価値のある外部情報 |
| `archive/` | 現行正本から外れた文書・履歴 |

エントリを追加・削除したら、必ず `INDEX.md` も更新する。

---

## 6. Issue起点の作業

Issue番号、Issue URL、「このissue」「issueベースで進める」などが示された場合は、Issue本文を確認してから作業に入る。

最低限の流れ:

- 着手前にIssue本文・状態・関連KBを読み、Statusを `In Progress` にする
- 背景・目的・Done条件が不足していれば、作業前にユーザーへ確認する
- 完了時はDone条件を照合し、作業ログと完了報告をIssueコメントへ残す
- stock化すべき決定事項だけをKBへ反映し、Statusを `Review` にする
- `Review -> Done` とIssue closeはユーザーのみが行う

詳細なStatus定義、着手前・完了時プロトコルは `patterns/issue_based_agent_workflow.md` を正本とする。

---

## 7. KB更新

KB更新前に同期する。

```bash
git -C ~/agent-kb pull
```

KBへの書き込み完了とは、ファイル更新、git commit、git push がすべて終わった状態を指す。更新・commitだけでは未完了。

コミットメッセージ例:

```text
add: tools/hubspot.md — APIキー設定の知識を追加
update: initiatives/foo.md — 方針を更新
```

---

## 8. KB更新タイミング

KB更新は `/close-issue <num>` コマンドの実行時に行う。Issue対応を通じて生まれた決定事項・設計判断・運用ルール・知見を、そのタイミングでまとめて抽出・記録する。

会話の各ターンで随時KBを更新する運用は廃止。

---

## 9. コラボレーション原則

- 問題は根本から直す。「気をつける」ではなく、守らざるを得ない仕組みにする。
- 不足、懸念、選択肢を先出しする。
- 完璧より動くものを優先し、試して改善する。
- 通常作業は確認してから動く。KBへのstock記録、振り返り促進、バッファ保存は自発的に行う。

---

## 10. バッファ運用

KBマスターへの書き込みやpushが失敗した場合は、`~/.kb_buffer/` 以下にKBと同じパス構造で保存し、`.kb_buffer/_BUFFER_LOG.md` に理由を記録する。

ユーザーが「バッファを突合して」と指示した場合は、バッファとKBマスターの差分を確認し、競合なしなら自動マージ、競合ありなら判断を仰ぐ。

---

## 11. セットアップ確認

このファイルを使い始める前に、以下を自分の環境に合わせて設定する。

1. **Section 2 のユーザー情報**をあなたの名前・会社・メールに書き換える
2. **`tools/github_projects.md`「安定ID」節**を取得して記録する（`patterns/api_usage_basics.md` §3 の確定値記録）。GitHub Projects 作成後に取得 query を1度回し、Project ID / Status field ID / Status option ID（Inbox〜Done）を表に転記する。
3. **`patterns/issue_based_agent_workflow.md`** 内のプレースホルダーを実際の値に書き換える:
   - `{YOUR_GITHUB_LOGIN}` → あなたのGitHubログイン名
   - `{PROJECT_NUMBER}` → GitHub Projectsの番号
   - `{PROJECT_ID}` → Project のnode ID（`gh api graphql` で取得）
   - `{STATUS_FIELD_ID}` → Status フィールドのID（同上）
   - `{IN_PROGRESS_OPTION_ID}` / `{REVIEW_OPTION_ID}` / `{DONE_OPTION_ID}` → Status各optionのID（手順2で取得した値）
4. **`~/.claude/commands/`** に `.claude/commands/` のファイルをコピーし、同じプレースホルダー（上記 option ID 含む）を置き換える
