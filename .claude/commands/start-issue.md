指定された Issue 番号の着手前プロトコルを実行する。

## 入力

`$ARGUMENTS` に Issue 番号が入る。

## 手順

### Step 1: Issue確認

```bash
gh issue view $ARGUMENTS --repo {YOUR_GITHUB_LOGIN}/agent-kb --json title,body,state,labels,comments,url
```

### Step 2: 状態確認

- Open/Close 状態を確認する
- Project Status を確認する（すでに Review / Done なら、着手していいか確認する）

### Step 3: 関連KB読み込み

本文 Metadata（`initiative_id`, `context`, `related_kb`）から関連 KB を特定して読む。

`~/agent-kb/INDEX.md` を参照して、関連するエントリがあれば作業前に読む。

`initiative:*` ラベルが付いていない場合は、`~/agent-kb/initiatives/_registry.md` の上位initiative一覧を確認し、Issue内容から「既存initiativeに紐づける / 新規initiativeとする / 紐づけなし」を推測して根拠とともに提示し、ユーザーに確認してからラベルを付与する。

**新規initiative作成の実行手順**（ユーザーが「新規initiativeとする」と確認した場合）:

1. `initiatives/_registry.md` の上位initiative一覧にエントリを追加する（`initiative_id`・表示名・status・正本doc）
2. GitHub ラベルを作成する

```bash
gh label create "initiative:<id>" --color 1d76db --description "<表示名>" --repo {YOUR_GITHUB_LOGIN}/agent-kb
```

3. Issueに `initiative:<id>` ラベルを付与する

```bash
gh issue edit $ARGUMENTS --repo {YOUR_GITHUB_LOGIN}/agent-kb --add-label "initiative:<id>"
```

4. 正本docが必要なら `initiatives/<id>.md` を作成し、`INDEX.md` にエントリを追加する
5. 変更をcommit・pushする

### Step 4: StatusをIn Progressに更新

記録値（安定キー）で楽観実行する。option IDを毎回取得する事前 query は挟まない（`~/agent-kb/patterns/api_usage_basics.md` §1「既知値で楽観実行→失敗時だけ確認」）。

記録値（`~/agent-kb/tools/github_projects.md`「安定ID」節。セットアップ時に取得して記録済みの値）:

| 項目 | 値 |
|---|---|
| Project ID | `{PROJECT_ID}` |
| Status field ID | `{STATUS_FIELD_ID}` |
| Status option `In Progress` | `{IN_PROGRESS_OPTION_ID}` |

IssueのProject item IDを取得する（item IDはissueごとに固定で記録対象外なので、これだけは取得する）。

```bash
gh api graphql -f query='
query($login:String!, $number:Int!){
  user(login:$login){ projectV2(number:$number){ items(first:100){ nodes{
    id
    content{ ... on Issue { number } }
  }}}}
}' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER} \
  --jq ".data.user.projectV2.items.nodes[] | select(.content.number == $ARGUMENTS) | .id"
```

StatusをIn Progressに設定する（記録値 `{IN_PROGRESS_OPTION_ID}` をそのまま使う）。

```bash
gh api graphql -f query='
mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
  updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){
    projectV2Item{ id }
  }
}' -F p={PROJECT_ID} -F i=<ITEM_ID> -F f={STATUS_FIELD_ID} -f o={IN_PROGRESS_OPTION_ID}
```

> **注意**: option IDの渡し方は `-f o=`（小文字・raw-field）を使う。`-F`（大文字）は型推論するため、純粋な10進数IDをintegerに変換してしまい `String!` 型エラーになる。

> **フォールバック（失敗時のみ）**: mutation が `option not found` 等で失敗したときだけ、下記 query で Status の全 option を再取得し、正しい ID で再実行する。あわせて `~/agent-kb/tools/github_projects.md` の Status option 表を新しい値で更新する（GitHub Projects の Status 選択肢を編集すると option ID は新規発行されるため）。
>
> ```bash
> gh api graphql -f query='
> query($login:String!, $number:Int!){
>   user(login:$login){ projectV2(number:$number){
>     fields(first:50){ nodes{
>       ... on ProjectV2SingleSelectField { id name options { id name } }
>     }}
>   }}
> }' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER} \
>   --jq '.data.user.projectV2.fields.nodes[] | select(.name == "Status") | .options[]'
> ```

### Step 5: 現状確認・着手

背景・目的・Done条件が不足していれば、作業前に確認する。

現在地、次アクション、確認事項を簡潔に提示してから作業を開始する。

### Step 6: 作業完遂後 — Review移行（完了時プロトコル）

`/start-issue` の責務は着手だけで終わらない。`In Progress → Review` の完了処理に専用コマンドは設けず、このコマンドで着手したIssueはそのまま Review への移行まで担う。

実作業が完了したら、明示の指示を待たず `~/agent-kb/patterns/issue_based_agent_workflow.md` §5 完了時プロトコル（Agent → Review）の手順1〜3を実行する。

1. Done条件を1つずつ照合する
2. Issueコメントに作業ログ・Done条件の照合結果を残す
3. StatusをReviewへ更新する（Step 4 と同じ要領で、記録値 Status option `Review` = `{REVIEW_OPTION_ID}` をそのまま使う。事前の option 取得 query は挟まない。失敗時のみ Step 4 のフォールバック query で再取得＋記録更新）

> **KB更新タイミング**: KBへのstock知識のcommit/pushは、ここ（Review移行時）では行わない。`/close-issue <num>`（ユーザーのレビュー後）のタイミングに統一する。
>
> ただし、Issueの成果物そのものがファイル変更である場合（ルール整備・ドキュメント修正など）、その変更のcommit/pushは「作業の一部」としてReview移行前に完了させる。これはstock知識の蒸留とは別物。
