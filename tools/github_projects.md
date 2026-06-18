# GitHub Projects

この文書は、GitHub Projects（v2）を `agent-kb` のIssue進行管理UIとして使うときの、**安定キーの記録場所**と**操作の落とし穴**の正本である。

横断的な操作思想（記録値で楽観実行→失敗時のみ再取得）は `patterns/api_usage_basics.md` を正とし、本文書はGitHub Projects固有の確定値と注意点だけを扱う。

> **セットアップ時に記録する**: 下の「安定ID」表の `{...}` を、自環境の GitHub Projects を作成したあとに取得して埋める。一度埋めれば、以降の Status 更新は毎回の取得 query を挟まず、この記録値で楽観実行する（[api_usage_basics.md](../patterns/api_usage_basics.md) §1）。

---

## 安定ID（記録値・最初に1度だけ埋める）

`/start-issue`・`/close-issue` のStatus更新は、まず**この記録値を使い**、GraphQLでの再取得を毎回挟まない。

| 種別 | ID |
|---|---|
| GitHub login | `{YOUR_GITHUB_LOGIN}` |
| Project number（URLの `/projects/<番号>`） | `{PROJECT_NUMBER}` |
| Project ID（グローバルNode ID） | `{PROJECT_ID}` |
| Status field ID | `{STATUS_FIELD_ID}` |

`Status` option ID（安定キー）:

| option | id |
|---|---|
| `Inbox` | `{INBOX_OPTION_ID}` |
| `Todo` | `{TODO_OPTION_ID}` |
| `In Progress` | `{IN_PROGRESS_OPTION_ID}` |
| `Blocked` | `{BLOCKED_OPTION_ID}` |
| `Review` | `{REVIEW_OPTION_ID}` |
| `Done` | `{DONE_OPTION_ID}` |

> **記録値の前提**: Project / field / option ID は安定キー。Project構成自体を変えない限り変動しない。まず本表の記録値を使い、`option not found` 系で失敗したときだけ下記の再取得 query を回し、本表を更新する。**毎回の事前再取得はしない。**
>
> **失敗時の再取得トリガー**: `updateProjectV2Field` で `Status` の選択肢を編集すると option ID は**新規発行**される（name/color/description を完全一致で渡しても既存IDは保持されない）。option IDを編集した直後は本表が古くなるので、必ず再取得して更新する。

item ID（IssueごとのProject item ID）はissueごとに固定だが記録対象外。Status更新のたびにそのissueの分だけ取得する（下記 手順1）。

---

## 取得方法（記録値を最初に埋める／失敗時に再取得する）

### Project ID / field ID / Status option ID をまとめて取得

```bash
gh api graphql -f query='
query($login:String!, $number:Int!){
  user(login:$login){ projectV2(number:$number){
    id title
    fields(first:50){ nodes{
      ... on ProjectV2SingleSelectField { id name options { id name } }
    }}
  }}
}' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER}
```

`Status` の `options { id name }` を上の表に転記する。`projectV2.id` が Project ID。

---

## Statusを更新する（記録値で楽観実行）

### 手順1: 対象IssueのProject item IDを取得

item IDはissueごとに固定だが記録対象外なので、対象issueの分だけ取得する。

```bash
gh api graphql -f query='
query($login:String!, $number:Int!){
  user(login:$login){ projectV2(number:$number){ items(first:100){ nodes{
    id
    content{ ... on Issue { number } }
  }}}}
}' -F login={YOUR_GITHUB_LOGIN} -F number={PROJECT_NUMBER} \
  --jq ".data.user.projectV2.items.nodes[] | select(.content.number == <ISSUE_NUMBER>) | .id"
```

### 手順2: Statusを設定（記録値の option ID をそのまま使う）

```bash
gh api graphql -f query='
mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
  updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){
    projectV2Item{ id }
  }
}' -F p={PROJECT_ID} -F i=<ITEM_ID> -F f={STATUS_FIELD_ID} -f o={IN_PROGRESS_OPTION_ID}
```

事前の option 取得 query は挟まない。`option not found` 等で失敗したときだけ、上の取得 query で再取得し記録値を更新してから再実行する。

---

## 落とし穴

### option IDが10進数のとき `-F o=` でString!型エラーになる

`updateProjectV2ItemFieldValue` で `singleSelectOptionId` に option ID を渡す際、`-F o=<ID>`（大文字）を使うと `gh` が型推論する。hex 文字列（例 `47fc9ee4`）はそのまま文字列扱いだが、**純粋な10進数文字列（例 `98236657`）は integer に変換されてしまい** `String!` 型エラーになる。

```
Variable $o of type String! was provided invalid value
```

**対処**: option ID は必ず `-f o=`（小文字・`--raw-field`）で渡す。`-f` は型推論をせず常に文字列として渡す。

```bash
# NG: -F は10進数をintegerに変換してしまう
-F o=98236657
# OK: -f は常にstring
-f o=98236657
```

### `updateProjectV2Field` は automation を無効化する

`updateProjectV2Field`（Status選択肢の編集等）を呼ぶと、GitHub Projects の automation（Auto-close issue 等）が自動的に無効化される。呼び出し後は手動で再有効化する。

- 導線: Project 本体（`https://github.com/users/<GH_USER>/projects/<PROJECT_NUMBER>`）を開く → 画面右上「…」メニュー → Workflows → 「Auto-close issue」を Enable。
- `.../projects/<PROJECT_NUMBER>/settings/workflows` を直接開くと **404** になるので使わない。

また `singleSelectOptions` は**全置換**の引数。既存選択肢を列挙し忘れると、その選択肢（とitem値）は失われる。選択肢の**削除**は破壊的なので、AIの自律実行にはせず承認制とする（[api_usage_basics.md](../patterns/api_usage_basics.md) §5）。

### `--jq` の出力破損

`gh api graphql ... --jq '...'` の整形出力に異物が混入し、field/item の値を誤判定することがある。**値の検証は `--jq` 整形を信用せず、生 JSON（`--jq` なし）でも裏取りする**（[api_usage_basics.md](../patterns/api_usage_basics.md) §7）。

---

## 認証スコープ

- Project v2の**読み取り**には `read:project`、**書き込み（field値設定）**には `project` scope が必要。
- 付与（非対話・デバイス認証フロー）:

```bash
gh auth refresh --hostname github.com -s project
```

- 確認: `gh auth status` の `Token scopes` に `project` が含まれること。スコープ不足で失敗したら不足分を提示する（[api_usage_basics.md](../patterns/api_usage_basics.md) §6）。

---

## Board / view の初期設定

- Status を列にした Board layout を使う。view（Board / Slice by / 保存view）の設定はAPIツールがなく、GitHub UIで行う。
- Issueメタデータ（initiative / type / priority）は label で管理する。label は Board の Slice by / Group by には使えない（GitHub仕様）。Filter / Group by には Labels を使える。
- 詳細な label 運用は `patterns/issue_based_agent_workflow.md` を参照。
