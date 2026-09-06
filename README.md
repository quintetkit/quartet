# Quartet

Claude Code に **設計者・実装者・レビュアー・コンフリクト解消担当** の 4 人格を
使い分けさせ、GitHub Issue とブランチを軸に並列開発を進めるための設定一式です。

コードは含みません。`CLAUDE.md`、`.claude/agents/`、`.github/ISSUE_TEMPLATE/` を
自分のプロジェクトにコピーして使います。技術スタックには依存しません。

## これが要る理由

Claude Code に大きめの依頼をすると、こうなりがちです。

- **頼んでいない範囲まで書き換わる。** 差分が膨らみ、レビューが追いつかなくなる
- **並列で走らせるとファイルが競合する。** あとから解きほぐせなくなる
- **進捗が会話の中にしかない。** 昨日どこで止まったか、履歴を遡らないと分からない
- **自分で書いて自分でレビューして自分でマージする。** 歯止めがない

原因は能力ではなく権限設計です。1 つの人格に全部やらせているから止まりません。

Quartet は責務を分け、**役割ごとに権限を落とします**。

- Coder は Issue に書かれた scope 外のファイルを変更できない
- main へマージできるのは Reviewer だけ
- コンフリクトの解消は専任の人格しかやらない
- 進捗は会話ではなく GitHub Issue のラベルに出る

## 4 つの人格

| 人格 | 役割 | コードを書くか |
|---|---|---|
| **Architect** | 依頼を Issue に分割。scope（担当パス）と依存を決め、並列着手できる組を作る | 書かない |
| **Coder** | 1 つの Issue だけを担当。自分の scope 内だけ変更して PR を出す | 書く |
| **Reviewer** | 受け入れ条件と scope 違反を確認してマージする。**マージできる唯一の人格** | 書かない |
| **Conflict Resolver** | コンフリクトが起きたときだけ登場し、解消して Reviewer に返す | 解消のみ |

```
依頼 → [Architect] Issue に分割（scope が重ならない組を作る）
         │
         ├─ [Coder] issue/12-xxx ─┐
         ├─ [Coder] issue/13-yyy ─┤ 並列
         └─ [Coder] issue/14-zzz ─┘
                                   │ PR ごとに1件ずつ
                                   ▼
                            [Reviewer] 受け入れ条件と scope を確認 → マージ
                                   │
                                   └─ コンフリクト → [Conflict Resolver] → Reviewer
```

## どこまでが機械で強制されているか

**「書かない」を、指示だけで担保していません。** ツールの割り当てで落としています。

| 人格 | `tools` | Write / Edit |
|---|---|---|
| Architect | `Bash, Read, Grep, Glob` | **持っていない** |
| Reviewer | `Bash, Read, Grep, Glob` | **持っていない** |
| Conflict Resolver | `Bash, Read, Edit, Grep, Glob` | Edit のみ |
| Coder | `*` | 持っている |

Architect と Reviewer は、**ファイルを編集する道具そのものを持っていません。**
「書かないでください」とお願いしているのではなく、書く手段がありません。

### 強制されていないもの

正直に書いておきます。**4 人格とも `Bash` を持っています。**

そのため、次の3つは**指示でしか担保されていません。**

- Coder が scope 外のファイルを触らないこと
- Coder がマージしないこと（`gh pr merge` は Bash から叩けます）
- Reviewer が実装しないこと（`Bash` からファイルは作れます）

`Bash` を外すと、テスト実行・`git`・`gh` が全部使えなくなり、
このワークフローは成立しません。**外せないので、代わりに境界を明示しています。**

守らせたい場合は、`.claude/settings.json` の
[`permissions.deny`](https://code.claude.com/docs/en/settings-reference) で
`Bash(gh pr merge:*)` のような規則を足せます。
ただし**セッション全体に効く**ので、Reviewer のマージも止まります。
人格ごとに Bash のサブコマンドを分ける手段は、いまのところありません。

**「機械で止まるもの」と「約束でしかないもの」を混ぜて説明しないのが、
このリポジトリの方針です。**

## 使い方

```bash
git clone https://github.com/quintetkit/quartet.git
cp -R quartet/CLAUDE.md quartet/.claude quartet/.github quartet/setup.sh /path/to/your-project/

cd /path/to/your-project
./setup.sh          # 状態管理に使うラベルを作成（gh CLI が必要）
```

`.github/workflows/issue-status.yml` も入っています。PR の動きに合わせて、
紐づく Issue の status ラベルを自動で張り替えます。

| PR で起きたこと | Issue のラベル |
|---|---|
| PR が作られた | `status:review` |
| レビューで changes requested | `status:changes-requested` |
| PR がマージされた | `status:done` |

PR 本文の `Closes #<番号>` から Issue を特定します。人格にラベル操作を任せると
付け忘れが必ず起きるので、状態管理は機械に寄せています。

あとは Claude Code にこう頼みます。

```
architect を使って、○○の機能を Issue に分割して
```

Issue が出来たら、並列着手グループの Coder を **1 メッセージで同時に起動** します。
順番に呼ぶと並列になりません。ここは `CLAUDE.md` に明記してあります。

## 設計上の判断

読む前に知っておくと納得しやすい点を 3 つ。

- **Reviewer のマージは並列にしない。** 実装は並列でも、マージは 1 件ずつ順に通します。
  ここを並列にするとコンフリクトが多発して、かえって遅くなります。
- **Coder は scope 外に出たら止まる。** 勝手に広げず、Architect への差し戻しとして報告します。
  「ついでに直しておきました」が起きないのはこのためです。
- **Conflict Resolver は呼ばれるまで動かない。** 先回りして予防的に解消しません。
  scope の設計ミスは Architect が直すべき問題で、解消で覆い隠すと原因が消えます。

## UI を含む開発について

この 4 人格構成は、UI の造形については何も言いません。そのため画面を作らせると、
生成 AI にありがちな既定形（全画面中央揃えのヒーロー、アイコン付き 3 カラムカード、
紫のグラデーション）がそのまま出てきます。

実装より先に UI 仕様を確定させる 5 人格目を挟むとこれを避けられます。
その UI Designer 人格と、Reviewer の判定基準、Issue 単位の並列実行スクリプト、
実践ガイド 10 章（日英）をまとめたものを **Quintet** として配布しています（[BOOTH で ¥4,980](https://quartet-dev.booth.pm/items/8807156)）。
実践ガイドだけなら [Zenn Book で ¥1,500](https://zenn.dev/quintetkit/books/claude-code-parallel-workflow) でも読めます（2章まで無料）。

UI Designer の評価軸は「人間が判断した跡が残っているか」の一点です。
グラデーション背景、純黒 `#000000` と純白 `#ffffff`、全画面中央揃えのヒーロー、
アイコン付き 3 カラム等幅カードといった生成 AI の既定形を名指しで禁止し、
状態は 8 つ（default / hover / focus-visible / active / disabled / loading / error / success）
すべて設計させます。

## 関連するツール

このワークフローは「**ファイルを共有する Issue は並列に走らせない**」を前提にしています。
それを機械で確かめるものを、別に MIT で公開しています。

- [scopecheck](https://github.com/quintetkit/scopecheck) — 開いている Issue のうち、
  対象範囲が重なっている組を、**重なっているファイルを名指しで**報告します
- [ccheck](https://github.com/quintetkit/ccheck) — `.claude/` の設定を検査します。
  指摘には必ず公式ドキュメントへの出典が付きます

どちらも依存ゼロで、インストールせずに試せます。

```bash
npx @quintetkit/ccheck        # リポジトリ直下で
npx @quintetkit/scopecheck --repo owner/name
```

## ライセンス

MIT

サブエージェント定義の書式は
[VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents)
を参考にしています。
