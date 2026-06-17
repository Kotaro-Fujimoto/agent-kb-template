ユーザーが提示したタイトルを起点に、GitHub Issue を対話的に整理してから起票する。

## 入力

`$ARGUMENTS` にタイトルが入る。

## 手順

### Step 1: 深掘りコミュニケーション

`$ARGUMENTS` のタイトルについて、以下の観点で深掘り質問をする。一度に全部聞かず、会話の流れに応じて必要な分だけ確認する。

- このIssueの背景・課題感は何か
- Done条件（何が完了したら閉じられるか）は何か
- 関連するinitiativeや既存Issueはあるか（`gh issue list --repo {YOUR_GITHUB_LOGIN}/agent-kb --state open` で一覧確認）
- type（task / decision / research / bug / chore）とpriority（high / medium / low）の目安

### Step 2: Issue本文ドラフトを提示・確認

対話で整理した内容をもとに以下の形式でドラフトを提示し、確認を得る。

```
## 背景
...

## Done条件
- [ ] ...

## Metadata
initiative_id: ...
context: ...
related_kb: [initiatives/foo.md](https://github.com/{YOUR_GITHUB_LOGIN}/agent-kb/blob/main/initiatives/foo.md)
```

### Step 3: 起票実行

確認が取れたら以下を順に実行する。

**a. Issue作成**

```bash
gh issue create \
  --repo {YOUR_GITHUB_LOGIN}/agent-kb \
  --title "<タイトル>" \
  --body "<本文>"
```

**b. GitHub Projectに追加**

Issue作成時に返ってくる node_id を使う。

```bash
gh api graphql -f query='
mutation($p:ID!,$i:ID!){
  addProjectV2ItemById(input:{projectId:$p,contentId:$i}){item{id}}
}' -F p={PROJECT_ID} -F i=<ISSUE_NODE_ID>
```

**c. StatusをInboxに設定**

StatusフィールドID: `{STATUS_FIELD_ID}`

Inboxのoption IDを取得してから設定する（IDは構成変更で変わりうるため毎回取得）。

```bash
gh api graphql -f query='
query($login:String!, $number:Int!){
  user(login:$login){ projectV2(number:$number){
    fields(first:50){ nodes{
      ... on ProjectV2SingleSelectField { id name options { id name } }
    }}
  }}
}' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER} \
  --jq '.data.user.projectV2.fields.nodes[] | select(.name == "Status") | .options[] | select(.name == "Inbox")'
```

取得したoption IDでStatusを設定する。

```bash
gh api graphql -f query='
mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
  updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){
    projectV2Item{ id }
  }
}' -F p={PROJECT_ID} -F i=<ITEM_ID> -F f={STATUS_FIELD_ID} -f o=<INBOX_OPTION_ID>
```

**d. ラベルを付与**

```bash
gh issue edit <num> --repo {YOUR_GITHUB_LOGIN}/agent-kb \
  --add-label "type:<t>,priority:<p>"
```

initiativeに紐づく場合は `initiative:<id>` も付与する。

### Step 4: 完了報告

起票したIssueのURLと番号を報告する。
