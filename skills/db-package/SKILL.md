---
name: db-package
description: >-
  PostgreSQL の業務ロジックを「スキーマ = 1 機能パッケージ」としてストアドプロシジャで
  カプセル化して開発するための手順書（iam / usedcar / workflow の 3 部作で実証した進め方）。
  新しい DB パッケージ・業務ロジックパッケージを作りたい・立ち上げたいとき、既存パッケージ
  （iam-component / usedcar-system / workflow-component やその同型リポジトリ）の続き・仕上げ・
  QA・外部レビュー・リリース準備をするとき、PL/pgSQL + pgTAP のパッケージ開発全般で必ず使う。
  「HANDOFF を読んで作業 #N から」「仕様 QA をやろう」「敵対的レビューをさせて」のような
  指示もこのスキルの対象。ユーザーが「スキル」と言わなくても該当タスクなら使うこと。
---

# DB パッケージ開発（スキーマ = 1 機能パッケージ）

3 パッケージ（iam / usedcar / workflow）の実証で確立した進め方。**コードの再利用より
「作り方と確かめ方の再利用」が価値の中心** — このスキルはその手順を渡す。

## 正とする場所（このスキルは複製しない。必要時にそこを読む）

| 何 | どこ |
|---|---|
| 設計規範（なぜ） | `~/workspaces/iam-component/docs/db-package-strategy.md`（特に §17 実証フィードバック）と `db-package-guidelines.md` |
| 振り返り（KPT） | `~/workspaces/iam-component/docs/db-package-retrospective.md` |
| **型紙（最新・第 3 世代）** | `~/workspaces/workflow-component/` — tools/ registries/ Makefile/ docker/ scripts/ CLAUDE.md の vendoring 元 |
| **SP 規約・SQL 書式・共通 SQL 雛形** | **このスキルの references/**（sp-conventions.md / sql-style.md / common-sql-patterns.md）が正。各パッケージの docs/conventions.md はここからのコピー + 題材固有分 |
| 連携の型紙 | workflow の docs/iam-contract.md（契約表 + contract test + soft reference） |

3 リポジトリを通読しないこと。型紙から必要箇所だけ生読みで写す。

### 前提の確認（セッション開始時に 1 回）

上の表のパスは発注者の作業環境（`~/workspaces/` 配下）を前提にしている。作業に入る前に
存在を確認し、無ければ**推測で進めず**次のとおりにする:

- 型紙（workflow-component）が無い: 立ち上げ #0 の vendoring 元が無いので、AskUserQuestion で
  所在（別パス / git URL / 無し）を確認する。「無し」なら #0 は型紙なしで進め、その旨と
  自作した骨組みの一覧を HANDOFF に明記する
- 設計規範・KPT（iam-component/docs）が無い: このスキルの「設計の既定値」を正として進める。
  規範の「なぜ」が必要な判断は ADR に判断根拠を自前で書く
- `uivolve-mock` スキルが無い: 仕上げ #8 の画面モックはスキップし、仕上げ報告に
  「画面モック: 未作成（uivolve-mock 不在）」と明記する。代替 DSL で作らない
- 既存パッケージの続きで HANDOFF.md が無い: 作業に入らず発注者に確認する

## 進め方の原則（全工程共通）

1. **HANDOFF 駆動**: セッションは対象リポジトリの HANDOFF.md を全部読むところから始める。
   HANDOFF §5 相当の決定済み事項は再議論しない — 変えたければ ADR を起こして発注者に確認
2. **docs/ が正**。設計と食い違う実装が必要になったら、実装せず ADR を起こす
3. **仕様書・設計文書は生読み**（native Read / sed）。圧縮ツール経由の読取りは識別子を壊し、
   iam では初期バグ 9 件中 6 件の原因になった
4. **発注者への質問は AskUserQuestion の選択式**で、推奨案を第一候補に置く。決定はそのまま ADR の素材
5. **1 作業 = 1 コミット**（レビュー 1 巡 = 1 コミット）。完了時は出口条件との突き合わせを報告
6. **「テストが green」と「穴がない」は別物**。仕上げでは必ず探針 QA と外部レビュー
   （references/qa-playbook.md）を回す

## 工程

作業は #0〜#8 の出口条件つき分割で進める。一覧と各出口条件は
**references/checklist.md** を読むこと（新規立ち上げ時・仕上げ #8 の開始時は必読）。
要点だけ:

- #0 で文書体系（doctype + validate）と開発環境（Docker + pgTAP + make）を型紙から vendoring
- **スコープ QA（FAQ 初版・デモ脚本・決定必須事項の ADR 化）は実装前に行う** —
  usedcar で実装後に回して手戻りした反省（workflow で解消済み）
- 実装（SP）は「UC ⇄ 公開関数 1:1・読み取りにも UC」「共通規約を conventions.md に先に書く」
  （conventions.md は **references/sp-conventions.md** をコピーして `<pkg>` / `XX` を置換し、§9 の欄を埋める）
- 仕上げ（#8）は個別ゲート: 性能目標つき bench / デモ + 期待出力 / FAQ 拡充 / SKILL.md・README /
  外部レビューの消化または明示受容 / 残っていた運用決定（保持方針など）の ADR 化

## 設計の既定値（3 例で収束した決定 — 新パッケージはこれを初期値にする）

- ERRCODE は 5 文字 SQLSTATE の独自プレフィクス（IAM01 / UCS01 / WF001 系）。DETAIL は診断用で
  契約は ERRCODE のみ
- 冪等キー: 全外部書込 SP で必須、テナントスコープ + **実行者束縛**、入力ハッシュは
  sha256(実行者 + 固有引数)。comment 等の注記引数は除外と明記。適用除外は ADR に列挙
- 時刻は `current_time()` ラッパ（**clock_timestamp() ベース・VOLATILE**。テストで差替可能）
- 状態遷移はステータスマスタ + 遷移許可テーブル（seed 固定）。**状態更新はヘルパ経由に限定**し、
  全変更経路が遷移表を通ることをレビュー対象にする
- grants は fail-closed: 再適用のたびに **PUBLIC 含む** REVOKE ALL → 明示 GRANT。
  追記専用テーブルは所有者からも UPDATE/DELETE/TRUNCATE を剥がす。配布状態はカタログからテスト
- イベントは outbox（配送バッファであって監査証跡ではない。保持期間 + クリーンアップ SP）
- AI エージェントは「人間の代理人」に一般化（専用接続ロール + 実行者の種別検証 + 判断根拠必須）
- パッケージ間連携は同一 DB + 公開 API 最小 + 契約テスト + soft reference（FK を張らない・
  表示名スナップショット）。提供側に API が足りなければ生テーブル参照で妥協せず追加してもらう
- 1 トランザクション = 1 書込 SP 呼出し。破壊的シグネチャ変更はしない（追加のみ）
- 同時実行制御は**悲観ロックが基本**（advisory lock + FOR UPDATE + 遷移表）。楽観ロック
  （`revision int` + `p_expected_revision` 必須、不一致は XX010）は**人間の編集セッションを挟む
  編集系エンティティにだけ**置く（DRAFT 定義・マスタ・自由記述）。状態遷移系には置かない。
  advisory lock キー列 `lock_no` は楽観ロックではない（2026-09-25 発注者決定）

## コーディング規約（references/ を読む — SKILL.md には要点のみ）

| 文書 | 決めること | 読むタイミング |
|---|---|---|
| **references/sp-conventions.md** | SP の外形: 引数順・検査順 8 段・ERRCODE 骨格 XX001〜009・冪等・ロール別 EXECUTE・監査/イベント・Tx・テナント境界 | 作業 #6 の冒頭。パッケージの conventions.md の雛形 |
| **references/sql-style.md** | コードの書き方: ファイル構成と番号帯、`_` / `p_` / `v_` 接頭辞、関数の定型、RAISE 書式、V ファイル規約（DROP 規約含む）、grants 骨格、pgTAP 規約、SP レビューチェックリスト | SP を書く直前・レビュー時 |
| **references/common-sql-patterns.md** | 000_common.sql / 900_grants.sql の雛形（時刻ラッパ・遷移検証・2 層ロック・冪等キー・監査/outbox・立場判定・agent 検証・fail-closed grants） | 作業 #0 の vendoring 時。型紙が無い環境ではここから起こす |

## テスト規約

- pgTAP は BEGIN...ROLLBACK で独立、fixture は公開 SP で毎回生成（固定 UUID 禁止）
- 権限マトリクスはカタログから検証（superuser では素通しになるため）
- 並行実行は 2 セッション bash（二重実行・競合・冪等キー競合を最低限含める）
- **弱いアサーション禁止**: ok(true) のプレースホルダ、件数だけの比較はしない。
  結果値・状態値まで比較する（第三者レビューで実際に露呈した失敗）

## QA・外部レビュー

仕上げ・QA 局面では **references/qa-playbook.md** を読んで実施する。中身: FAQ 駆動 QA /
探針シナリオ QA / 外部 AI レビューの分割方法（観点×対象、方向別の敵対的レビュー 3 種）/
指摘の 4 分類記録（修正・文書化・テスト追加・理由つき明示受容）。
