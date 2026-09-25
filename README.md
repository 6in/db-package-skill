# db-package-skill

PostgreSQL の業務ロジックを「スキーマ = 1 機能パッケージ」としてストアドプロシジャで
カプセル化して開発するための手順書スキル（PL/pgSQL + pgTAP）。
iam / usedcar / workflow の 3 パッケージで実証した進め方を、Claude Code などの
AI エージェント向けスキルとして配布する。

## 中身

```
.claude-plugin/
  plugin.json          # Claude Code プラグインマニフェスト
  marketplace.json     # このリポジトリ自身をマーケットプレイスとして公開する定義
skills/
  db-package/
    SKILL.md           # スキル本体（原則・工程・設計の既定値・テスト規約）
    references/
      checklist.md            # 作業 #0〜#8 の出口条件と仕上げゲート
      qa-playbook.md          # FAQ 駆動 QA / 探針シナリオ QA / 外部 AI レビューの回し方
      sp-conventions.md       # SP 共通規約の雛形（引数順・検査順・ERRCODE・冪等・ロール・Tx）
      sql-style.md            # SQL / PL/pgSQL コーディング規約とレビューチェックリスト
      common-sql-patterns.md  # 000_common.sql / 900_grants.sql の雛形（<pkg> プレースホルダ）
docs/
  research/                   # 外部レビュー記録（配布物ではない）
```

規約 3 本は**このリポジトリが正**。各パッケージの `docs/conventions.md` はここからコピーして
題材固有分（ERRCODE の意味・イベント語彙・適用除外）を足す。共通部分の変更はこちらに戻す。

## インストール

### Claude Code（プラグインとして）

```bash
# GitLab から
claude plugin marketplace add https://gitlab.nasa.future.co.jp/fre/components/db-package-skill.git
claude plugin install db-package@db-package-skill

# ローカルクローンから試す場合
claude plugin marketplace add /path/to/db-package-skill
claude plugin install db-package@db-package-skill
```

更新は `claude plugin update db-package`。

インストール後はスキル名 `db-package:db-package` として読み込まれる。
`~/.claude/skills/db-package` に同名の個人スキルを置いている場合は重複するので、
どちらか一方に寄せること。

### skills CLI（Claude Code / Codex / Cursor など共通）

```bash
npx skills add https://gitlab.nasa.future.co.jp/fre/components/db-package-skill.git
```

### 手動

```bash
cp -r skills/db-package ~/.claude/skills/
```

## 使い方

スキルは自動で発火する。次のような依頼が対象になる。

- 新しい DB パッケージ・業務ロジックパッケージを立ち上げたい
- 既存パッケージ（iam-component / usedcar-system / workflow-component など）の続き・仕上げ・QA・リリース準備
- 「HANDOFF を読んで作業 #N から」「仕様 QA をやろう」「敵対的レビューをさせて」

## 前提（外部依存）

このスキルは手順書であり、次の資産は同梱していない。

| 依存 | 用途 | 無い場合の振る舞い（SKILL.md「前提の確認」） |
|---|---|---|
| `~/workspaces/workflow-component` | 立ち上げ #0 の型紙（tools / registries / Makefile / docker） | 所在を発注者に確認。無ければ型紙なしで骨組みを自作し HANDOFF に明記。共通 SQL は `common-sql-patterns.md` から起こせる |
| `~/workspaces/iam-component/docs/` | 設計規範・振り返り（判断の「なぜ」） | SKILL.md の「設計の既定値」を正として進める |
| 個人スキル `uivolve-mock` | 仕上げ #8 の画面モック | モックをスキップし仕上げ報告に明記 |
| 作業対象リポジトリの `HANDOFF.md` | セッション開始点 | 作業に入らず発注者に確認 |

別環境で使う場合は SKILL.md の「正とする場所」の表を自分の配置に書き換える。
型紙リポジトリの版は固定していない（社内の同一ワークスペース前提）。配布範囲を広げる
場合は型紙の同梱か git URL + ref の固定を検討すること。

## 検証

```bash
claude plugin validate . --strict
claude plugin validate skills --strict
```

## 更新の作法

- SKILL.md / references を変えたら `.claude-plugin/plugin.json` と `marketplace.json` の
  `version` を上げる
- 実証で得た知見はスキル側（この repo）に集約し、各パッケージリポジトリには複製しない
