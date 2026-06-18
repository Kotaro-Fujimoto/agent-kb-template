# Issue-Based Agent Workflow

この文書は、AIエージェントがGitHub Issueを起点に作業するための恒常的な運用ルールである。

導入プロジェクトの経緯ではなく、今後も使う作業パターンとして扱う。

各プロトコルはスラッシュコマンドとして実装されている。コマンド定義: `~/.claude/commands/`

API・GitHub Projects操作の横断的な基本動作（記録値で楽観実行→失敗時だけ再取得）は `patterns/api_usage_basics.md`、GitHub Projects固有の安定キー・落とし穴は `tools/github_projects.md` を正本とする。

| コマンド | 用途 |
|---|---|
| `/file-issue <タイトル>` | Issue起票（対話でDone条件・背景を整理してから起票） |
| `/start-issue <num>` | 着手前プロトコル（Issue確認・関連KB読込・Status=In Progress） |
| `/close-issue <num>` | 完了処理（KB更新・完了コメント・Status=Done・Issue close） |

---

## セットアップ時に書き換えるプレースホルダー

以下の値をあなたの環境の実際の値に置き換える。

| プレースホルダー | 説明 | 取得方法 |
|---|---|---|
| `{YOUR_GITHUB_LOGIN}` | GitHubログイン名 | `gh api user --jq .login` |
| `{PROJECT_NUMBER}` | GitHub Projectsの番号（URLの `/projects/<番号>`） | GitHub Projects URL |
| `{PROJECT_ID}` | ProjectのグローバルNode ID | `gh api graphql -f query='query{user(login:"{YOUR_GITHUB_LOGIN}"){projectV2(number:{PROJECT_NUMBER}){id}}}'` |
| `{STATUS_FIELD_ID}` | StatusフィールドのID | `gh api graphql ...` でフィールド一覧を取得 |
| `{IN_PROGRESS_OPTION_ID}` / `{REVIEW_OPTION_ID}` / `{DONE_OPTION_ID}` ほか | Status各optionのID（コマンドのStatus更新で使う） | `tools/github_projects.md`「安定ID」節の取得 query で全option一括取得 |

Status option ID は `tools/github_projects.md`「安定ID」節に**記録するのが正本**。一度取得して記録すれば、`/start-issue`・`/close-issue` は毎回の option 取得 query を挟まず、記録値で楽観実行する（`patterns/api_usage_basics.md` §1）。

---

## 1. 基本思想

Issueは、未解決の問い・作業・判断・不具合・改善案を扱うフローの正本である。

KBは、Issueでの作業から生まれた決定事項・設計判断・運用ルールを蒸留するストックの正本である。

役割分担:

| 場所 | 役割 |
|---|---|
| Issue本文 | 背景、目的、Done条件、Metadata |
| Issueコメント | 作業ログ、完了報告、Done条件の照合結果 |
| Project Status | 進行状態 |
| KB | 現在有効な方針、設計判断、運用ルール、背景 |

進行状況をKBへ転記しない。Open/Close、Status、残タスク、完了報告はIssue側を正本とする。

initiative docの「現在地」セクションと `*_history.md` に残してよい粒度は `kb_document_design.md`「現在地・history_logの粒度」を正本とする。

---

## 2. Status定義

| Status | 意味 | 主に動かす主体 |
|---|---|---|
| Inbox | 起票直後・未仕分け | 人 / Agent |
| Todo | 仕分け済み・着手可能 | 人 / Agent |
| In Progress | 作業中 | Agent |
| Blocked | 依存・外部要因・人間判断待ちで停止中 | 人 / Agent |
| Review | Done条件を満たしたとAgentが報告し、人間レビュー待ち | Agent |
| Done | 人間が確認しclose | 人 |

基本遷移:

```text
Inbox -> Todo -> In Progress -> Review -> Done
                  ^
                  |
               Blocked
```

Agentの自律範囲は `Inbox -> Todo -> In Progress -> Blocked -> Review` まで。`Review -> Done` とIssue closeは `/close-issue <num>` コマンドで実行する（ユーザーがレビュー完了後に打つ）。

---

## 3. 着手前プロトコル

Issue番号、Issue URL、「このissue」「issueベースで進める」などが示された場合は、Issue本文を確認してから作業に入る。

手順:

1. Issue本文を読む
2. Open/Close状態とProject Statusを確認する
3. Metadata、label、本文から関連KBを特定して読む
4. 着手するIssueのStatusを `In Progress` に更新する
5. 背景・目的・Done条件が不足していれば、作業前にユーザーへ確認する
6. 現在地、次アクション、確認事項を簡潔に提示してから進める

CLI経由でGitHubを扱える環境では、Issue本文の確認に `gh` を使う。

```bash
gh issue view <番号> --repo {YOUR_GITHUB_LOGIN}/agent-kb --json title,body,state,labels,comments,url
```

正規経路でIssue本文を読めない場合は、`curl`、web検索、公開GitHub APIへ逃げず、読めない旨を報告して止まる。

---

## 3.1 initiative未紐づけIssueへの対応

着手前に `initiative_id` ラベルが付いていない場合、以下の手順で推測・確認する。

**分割サイン（initiative化を検討するタイミング）**:

以下のいずれかに当てはまれば、initiative化を積極的に提案する:

1. **着手前に「何から始めるか」を選ぶ必要がある** — 独立した入口が複数ある
2. **`type:*` ラベルを複数つけられそうな内容** — 性質の異なる作業（research / task など）が混在している

Done条件の数は判断軸にしない（複数のDone条件を持つissueは適切な書き方であって分割サインではない）。

分割サインの検知タイミング: `/file-issue` 起票時・`/start-issue` 着手時・`/close-issue` 完了時。

**3択の判断軸**:

| 状況 | 推測 |
|---|---|
| Issue内容が既存の上位initiativeのスコープに明らかに合致する | 既存initiativeに紐づける案を提示 |
| 複数Issueにまたがる見込みの新テーマで、独立した追跡軸が必要 | 新規initiative提案 |
| 一時的・単発・どのinitiativeにも自然に属さない | 紐づけなし提案 |

**手順**:

1. `initiatives/_registry.md` の上位initiative一覧を確認する
2. Issue内容・タイトルから最も近いinitiativeを推測し、根拠とともに3択で提示する
3. ユーザーの確認を取ってから、ラベル付与または新規initiative作成を実施する

確認前にラベルを付与しない。推測が外れるコストより、確認の手間のほうが小さい。

**新規initiative作成の実行手順**:

ユーザーが「新規initiativeとする」と確認した場合、以下を実施する。

1. `initiatives/_registry.md` の上位initiative一覧にエントリを追加する（`initiative_id`・表示名・status・正本doc）
2. GitHub ラベルを作成する

```bash
gh label create "initiative:<id>" --color 1d76db --description "<表示名>" --repo {YOUR_GITHUB_LOGIN}/agent-kb
```

3. Issueに `initiative:<id>` ラベルを付与する

```bash
gh issue edit <num> --repo {YOUR_GITHUB_LOGIN}/agent-kb --add-label "initiative:<id>"
```

4. 正本docが必要なら `initiatives/<id>.md` を作成し、`INDEX.md` にエントリを追加する。docを作成する場合は本文に「## 退場条件」セクションを必ず書く（`initiatives/` 配下の全docタイプに適用。トリガー＋行き先のフォーマット正本: `initiatives/_registry.md`「退場条件」節）
5. `_registry.md`（と `INDEX.md`）の変更をcommit・pushする

---

## 3.2 Review後の追加作業

Issueが Review 状態に入った後に追加の作業が発生した場合、完了後に自動でIssueコメントを残す。明示的な指示を待たない。

コメントに含める内容:
- 何を追加したか
- なぜ追加したか（指摘・気づき・追加要件）

---

## 4. 作業中のルール

- 背景やDone条件を勝手に補完しない
- 新しい未決事項が出たらIssue化する
- Issueが大きくなりすぎたら、親Issueに索引と共通方針を残し、子Issueへ分割する
- 着手前に解消すべき確認事項は、実作業Issue内に埋めず `type:decision` または `type:research` の独立Issueにする
- KBへは作業ログではなく、stock化すべき決定事項だけを反映する

---

## 5. 完了時プロトコル

作業完了時（Agent → Review）:

1. Done条件を1つずつ照合する
2. Issueコメントに作業ログ・Done条件の照合結果を残す
3. Statusを `Review` へ上げる

ユーザーのレビュー後（`/close-issue <num>` で実行）:

4. stock化すべき決定事項、設計判断、正本だけをKBへ反映する
5. 完了報告・KB更新内容をIssueコメントへ残す
6. StatusをDone・Issue closeまで完了する

KBに書くもの:

- 決定事項
- 現在有効な設計判断
- 運用ルール
- 判断理由として必要な圧縮された背景

Issueコメントに書くもの:

- 何を実施したか
- Done条件をどう満たしたか
- 関連commit
- 残る確認事項

---

## 6. Metadataとlabel

Issue本文のMetadataは、AI向け再開文脈の正とする。

代表項目:

- `initiative_id`
- `context`
- `related_kb`: KBファイルへのリンク。GitHub UIから直接クリックできるよう、Markdownリンク形式で書く。
  例: `[initiatives/foo.md](https://github.com/{YOUR_GITHUB_LOGIN}/agent-kb/blob/main/initiatives/foo.md)`

GitHub labelは、人間向けビュー・フィルタ軸として使う。

| label | 用途 |
|---|---|
| `initiative:<id>` | 上位initiativeへの紐付け |
| `type:<kind>` | task / decision / research / bug / chore |
| `priority:<level>` | high / medium / low |

`initiative:*` は `initiatives/_registry.md` の上位initiative `initiative_id` と一致させる。横断Issueでは複数付与してよい。

GitHub Projects / labels / GraphQL操作の詳細は `tools/github_projects.md` を正本とする。

---

## GitHub Projects GraphQL 操作リファレンス

詳細な安定キーの記録・落とし穴・再取得トリガーは `tools/github_projects.md` を正本とする。ここでは Status 更新の共通パターンだけ示す。

### Status を更新する（記録値で楽観実行）

option IDは `tools/github_projects.md`「安定ID」節の記録値（プレースホルダー置換済み）をそのまま使う。**毎回の option 取得 query は挟まない**（`patterns/api_usage_basics.md` §1）。失敗時のみ再取得する。

```bash
# IssueのProject item IDを取得（item IDはissueごとに固定で記録対象外）
gh api graphql -f query='
query($login:String!, $number:Int!){
  user(login:$login){ projectV2(number:$number){ items(first:100){ nodes{
    id
    content{ ... on Issue { number } }
  }}}}
}' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER} \
  --jq ".data.user.projectV2.items.nodes[] | select(.content.number == <ISSUE_NUMBER>) | .id"

# Statusを設定（記録値の option ID をそのまま使う。例: In Progress）
gh api graphql -f query='
mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
  updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){
    projectV2Item{ id }
  }
}' -F p={PROJECT_ID} -F i=<ITEM_ID> -F f={STATUS_FIELD_ID} -f o={IN_PROGRESS_OPTION_ID}
```

> **注意**: option IDの渡し方は `-f o=`（小文字・raw-field）を使う。`-F`（大文字）は型推論するため、純粋な10進数IDをintegerに変換してしまい `String!` 型エラーになる。
>
> **失敗時のみ再取得**: `option not found` 等で失敗したときだけ、Status の全 option を再取得し（`tools/github_projects.md` の取得 query）、`tools/github_projects.md` の記録値を更新してから再実行する。
