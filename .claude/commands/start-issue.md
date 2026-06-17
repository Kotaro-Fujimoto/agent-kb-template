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

### Step 4: StatusをIn Progressに更新

StatusフィールドID: `{STATUS_FIELD_ID}`

In Progressのoption IDを取得する。

```bash
gh api graphql -f query='
query($login:String!, $number:Int!){
  user(login:$login){ projectV2(number:$number){
    fields(first:50){ nodes{
      ... on ProjectV2SingleSelectField { id name options { id name } }
    }}
  }}
}' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER} \
  --jq '.data.user.projectV2.fields.nodes[] | select(.name == "Status") | .options[] | select(.name == "In Progress")'
```

IssueのProject item IDを取得する。

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

StatusをIn Progressに設定する。

```bash
gh api graphql -f query='
mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
  updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){
    projectV2Item{ id }
  }
}' -F p={PROJECT_ID} -F i=<ITEM_ID> -F f={STATUS_FIELD_ID} -f o=<IN_PROGRESS_OPTION_ID>
```

> **注意**: option IDの渡し方は `-f o=`（小文字・raw-field）を使う。`-F`（大文字）は型推論するため、純粋な10進数IDをintegerに変換してしまい `String!` 型エラーになる。

### Step 5: 現状確認・着手

背景・目的・Done条件が不足していれば、作業前に確認する。

現在地、次アクション、確認事項を簡潔に提示してから作業を開始する。
