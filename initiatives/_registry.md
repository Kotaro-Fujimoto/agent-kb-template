# Initiative registry

最終更新: （セットアップ時に記入）

## 目的

KB `initiatives/` 配下のファイルを、GitHub Issues / Projects で扱える安定した単位に整理する。

GitHub issueの主文脈は、ファイルパスではなく上位initiativeの `initiative_id` に紐づける。

## 運用ルール

- Initiativeの紐付けは GitHub label `initiative:<initiative_id>` で行う。
- label名は下表の `initiative_id` と一致させる。
- 子initiative、設計書、履歴、作業メモは issue本文 `Metadata` の `context`/`related_kb` リンクで補足する。
- 複数initiative横断issueでは、`initiative:*` ラベルを複数付与してよい。
- 物理的なファイル移動・リネームはリンク切れリスクがあるため、このregistry運用が安定してから判断する。
- archive配下の完了済みinitiativeは、このregistryの対象外。必要になったら別途 `archive/_registry.md` を作る。

## doc_type

| doc_type | 意味 |
|---|---|
| `top_level_initiative` | GitHub Projectsの Initiative 軸になる上位initiative |
| `child_initiative` | 上位initiative配下の独立した作業テーマ・サブプロジェクト |
| `design_doc` | 設計・運用・アーキテクチャなど、実装や判断の正本になる文書 |
| `history_log` | 過去の経緯・実行ログを退避した履歴文書 |
| `session_note` | 特定セッション・ミーティング・中断地点の作業メモ |
| `deprecated` | 廃止・更新停止・別正本に移管済みの文書 |

## status

| status | 意味 |
|---|---|
| `active` | 進行中、または現在も判断・作業の起点として読む |
| `planning` | 検討中で、実運用・実装の方針がまだ固まりきっていない |
| `paused` | 再開可能だが、現時点では作業が止まっている |
| `reference` | 現在作業の正本ではないが、経緯確認のため参照する |
| `deprecated` | 更新停止。別の正本または後継文書を参照する |

## `initiative:*` ラベルの採番元（上位initiative一覧）

<!-- 新しい上位initiativeを足したら、対応する initiative:<id> ラベルを gh label create で作る -->

| initiative_id | 表示名 | status | 正本doc | 備考 |
|---|---|---|---|---|
| （ここに追加する） | | | | |

## ドキュメント棚卸し

| ファイル | initiative_id | parent | doc_type | status | 位置づけ |
|---|---|---|---|---|---|
| （ここに追加する） | | | | | |

## 新規initiative作成手順

新しい上位initiativeを作成する際は、以下の手順を実施する。

**1. 本ファイル（`_registry.md`）に追加する**

「`initiative:*` ラベルの採番元（上位initiative一覧）」表に1行追加する:

```
| `<initiative_id>` | <表示名> | `active` | [<id>.md](<id>.md) | <備考> |
```

**2. GitHub ラベルを作成する**

```bash
gh label create "initiative:<id>" --color 1d76db --description "<表示名>" --repo {YOUR_GITHUB_LOGIN}/agent-kb
```

**3. 正本docを作成する（任意）**

`initiatives/<id>.md` を作成し、背景・目的・確定方針を書く。作成した場合は `INDEX.md` にもエントリを追加する。

**4. 変更をcommit・pushする**

```bash
git -C ~/agent-kb add initiatives/_registry.md INDEX.md initiatives/<id>.md
git -C ~/agent-kb commit -m "add: initiatives/_registry.md — <id> initiativeを追加"
git -C ~/agent-kb push
```

---

## アーカイブ時の運用ルール

initiative KBを `archive/initiatives/` へ移動する際、以下の手順を実施する。

**1. GitHub ラベルをリネームする**

```bash
gh label edit "initiative:<id>" --name "~archived:<id>" --repo {YOUR_GITHUB_LOGIN}/agent-kb
```

- `~` プレフィックスにより、GitHub Projects の Slice by label で一覧の最下部に表示される。
- リネームは全issueに自動反映されるため、過去issueとの紐づきは維持される。

**2. 本ファイル（`_registry.md`）を更新する**

- 上位initiative一覧の表からそのエントリを削除する。
- ドキュメント棚卸し表からそのエントリを削除する。
- 棚卸し表下部の注釈行に `> YYYY-MM-DD（#<issue>）: <id> をアーカイブ済みのため削除。` を追記する。
