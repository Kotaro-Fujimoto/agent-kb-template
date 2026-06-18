ユーザーが内容を確認・承認したIssueの完了処理を実行する。KB更新・完了コメント・StatusをDone・Issue closeまでを担う。

## 入力

`$ARGUMENTS` に Issue 番号が入る。

## 前提

このコマンドはユーザーがレビュー済みの Issue に対して打つ。実行 = 完了承認とみなしてよい。

## 手順

### Step 1: Issue確認

```bash
gh issue view $ARGUMENTS --repo {YOUR_GITHUB_LOGIN}/agent-kb --json title,body,state,labels,comments,url
```

### Step 2: Done条件の照合

Issue本文の Done条件を1つずつ照合し、満たしているかを確認する。未達のものがあれば報告して止まる。

### Step 3: KB更新

今回の作業・会話を振り返り、KB にストックすべき内容を抽出する。

抽出対象：
- 決定事項・設計判断・運用ルール
- ツール・API・システムの知見、操作の再現性を高めるためのもの
- 組織・事業のコンテキスト
- 繰り返し使える判断パターン

更新手順：

```bash
git -C ~/agent-kb pull
```

対象ファイルを更新し、commit・push まで完了させる。

```bash
git -C ~/agent-kb add <ファイル>
git -C ~/agent-kb commit -m "update: <path> — <内容の要約>"
git -C ~/agent-kb push
```

KB への記録が不要な場合（ストック候補がない）は、その旨をコメントに明記する。

### Step 4: 完了コメントをIssueに残す

```bash
gh issue comment $ARGUMENTS --repo {YOUR_GITHUB_LOGIN}/agent-kb --body "..."
```

コメントに含める内容：
- 何を実施したか
- Done条件をどう満たしたか
- KB更新した場合は対象ファイルと commit hash
- 残る確認事項（あれば）

### Step 5: StatusをDoneに設定

記録値（安定キー）で楽観実行する。option IDを毎回取得する事前 query は挟まない（`~/agent-kb/patterns/api_usage_basics.md` §1「既知値で楽観実行→失敗時だけ確認」）。

記録値（`~/agent-kb/tools/github_projects.md`「安定ID」節。セットアップ時に取得して記録済みの値）:

| 項目 | 値 |
|---|---|
| Project ID | `{PROJECT_ID}` |
| Status field ID | `{STATUS_FIELD_ID}` |
| Status option `Done` | `{DONE_OPTION_ID}` |

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

StatusをDoneに設定する（記録値 `{DONE_OPTION_ID}` をそのまま使う）。

```bash
gh api graphql -f query='
mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
  updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){
    projectV2Item{ id }
  }
}' -F p={PROJECT_ID} -F i=<ITEM_ID> -F f={STATUS_FIELD_ID} -f o={DONE_OPTION_ID}
```

> **注意**: option IDの渡し方は `-f o=`（小文字・raw-field）を使う。`-F`（大文字）は型推論するため、`98236657` のような純粋な10進数IDをintegerに変換してしまい `String!` 型エラーになる。

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

### Step 6: initiative化の提案（任意）

以下のいずれかに当てはまる場合、Issue closeの前にinitiative化を提案する:

- このIssueから続きそうな関連テーマが浮かんでいる
- 今回のDone条件に性質の異なる作業（research / task など）が混在していた
- 「次はこれをやりたい」という関連テーマが自然と出てきた

提案例: 「このテーマは今後も発展しそうです。initiativeとして `_registry.md` に登録しますか？」

ユーザーが「initiative化する」と判断した場合は、`patterns/issue_based_agent_workflow.md`「新規initiative作成の実行手順」に従って作成してからcloseする。

### Step 7: 完了報告

KB更新内容・Issueコメント・close完了を一行ずつ報告する。

> **注意**: Status=Done への更新で GitHub Projects の「Auto-close issue」automation が自動的に Issue を close する。`gh issue close` は不要。
