# 外部レビュー記録 2026-09-25 — Codex 敵対的レビュー（初回パッケージング時）

- レビュア: OpenAI Codex（codex プラグイン経由、セッション 01a0d871-22bf-7780-81f4-49a9b3d307f8）
- 依頼方式: 1 系統（リポジトリ全体・読み取り専用・敵対的）。対象は `.claude-plugin/` /
  `skills/db-package/` / `README.md`
- 依頼観点: manifest / frontmatter の妥当性、ファイル間参照、矛盾・曖昧な指示、
  実 DB に対する危険な手順、README と実体の不一致
- 結果: Critical 0 / High 2 / Medium 2 / Low 1。総評「構造は健全だが無人運用にはそのまま使えない」
- Codex 側で未実施: `claude plugin validate`（サンドボックスで CLI 実行不可）。
  こちらで `--strict` 通過を確認済み

## 指摘の採否（4 分類）

| # | 重大度 | 指摘 | 分類 | 処置 |
|---|---|---|---|---|
| 1 | High | 探針 QA の「実 DB で実行」に接続先確認・使い捨て DB の要件がなく、共有 DB で恒久ロックアウト等を起こしうる | **文書化** | qa-playbook.md §2 に「実 DB = make up の Docker DB」「接続先確認」「合成 fixture のみ」「探針後は DB を捨てる」を追記 |
| 2 | High | 「DEFAULT 付き引数の末尾追加」は CREATE OR REPLACE では別オーバーロードになり旧呼出が曖昧になる。型紙 workflow-component も DROP を持たない（34 本すべて CREATE OR REPLACE のみ）ことを確認 | **文書化** | checklist.md「完了後の保守サイクル」に旧シグネチャ DROP（または互換ラッパ）・grants 再適用・アップグレードテストを追記 |
| 3 | Medium | 型紙リポジトリと uivolve-mock が外部依存で、取得手順・版固定・欠如時の分岐がない | **文書化** + **明示受容（版固定）** | SKILL.md に「前提の確認」節（欠如時の分岐 4 種）、README に依存表を追加。**版固定は行わない**: 社内同一ワークスペース前提の配布で、型紙は継続的に育てる資産のため ref 固定は逆に陳腐化を招く。配布範囲を広げる時に再検討（README に明記） |
| 4 | Medium | 「各巡の後は必ず全ゲート再実行（test/validate/demo/bench）」が実装前の巡に適用不能 | **文書化** | qa-playbook.md §5 を工程別ゲートに書き換え。未到達ゲートは「該当工程未到達」と記録するよう明記 |
| 5 | Low | checklist.md 39 行目の ```uivolve がコードフェンスとして開き、以降が全てコード扱いで描画される | **修正** | インラインコードに変更。フェンス開閉なしを確認 |

## 指摘なしと確認された項目

manifest の JSON 妥当性、プラグイン名・版・source と README の一致、frontmatter、
references の存在、認証情報・自動実行フックの不在。

## 次巡への持ち越し

- 型紙側（workflow-component）にも #2 の DROP 規約を反映するかは別リポジトリの作業（このスキルの範囲外）
- 本記録はスキルの配布物ではない（`docs/` はプラグイン読み込み対象外）
