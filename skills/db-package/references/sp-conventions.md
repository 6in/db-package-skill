# SP 共通規約（雛形） — 作業 #6 の冒頭でパッケージの docs/conventions.md にコピーして埋める

3 パッケージ（iam / usedcar / workflow）で収束した公開ストアドプロシジャ（SP）の外形規約。
**この文書が正**で、各パッケージの `docs/conventions.md` はここからコピーし、`<pkg>` と
`XX` を置換して題材固有の項目（ERRCODE の意味・イベント語彙・適用除外）を埋めたもの。
共通部分を変えたいときはパッケージ側でなくこの雛形を直す（ADR を添える）。

記法: `<pkg>` = スキーマ名（例 `wf`）、`XX` = ERRCODE プレフィクス 2〜3 文字（例 `WF`、`IAM`、`UCS`）。
`<pkg>_owner` などのロール名は「ロール」節を参照。

## 1. 引数順

```
書込 SP: p_tenant_id uuid, p_actor_id uuid, ...必須の固有引数..., p_idem_key text, ...任意の固有引数（DEFAULT 付き）...
読取 SP: p_tenant_id uuid, p_actor_id uuid, ...絞り込み..., p_limit int DEFAULT 100, p_offset bigint DEFAULT 0
代理 SP: p_tenant_id uuid, p_actor_id uuid, p_on_behalf_of uuid, p_decision_basis text, ...以下書込 SP と同じ...
```

- `p_idem_key` は**必須群の末尾・任意群の前**（PostgreSQL は DEFAULT 付き引数の後ろに必須引数を置けない）
- `p_actor_id` = 実行者（acted_by）。真正性の担保はアプリ層の責務（DB は認可境界であって認証境界ではない）
- 代理 SP は名義人 `p_on_behalf_of` と判断根拠 `p_decision_basis` を固有引数の先頭に取る。
  人間の代理人も AI エージェントも同じ SP を使う（「AI = 人間の代理人」への一般化）
- 読取 SP は冪等キーを取らない。`p_limit` は 1 以上、`p_offset` は 0 以上。NULL・範囲外は XX001
- comment / reason / decision_basis のような**監査注記の引数は操作の同一性を構成しない**
  （冪等ハッシュの対象外 — §4）

## 2. 検査順（全書込 SP）

| 段 | 検査 | ヘルパ | ERRCODE |
|---|---|---|---|
| 1 | 入力形式（NULL・型・値域・必須属性） | — | XX001 |
| 2 | actor（と on_behalf_of）の有効性・テナント所属（表示名スナップショットもここで取得） | `<pkg>._principal()` | XX002 |
| 3 | 冪等キー開始（同一キーを advisory lock で直列化。既実行なら保存結果を再返却、同一キー + 異入力は競合） | `<pkg>._idem_begin()` | XX007 |
| 4 | ロック取得（集約単位 or テナント単位。**キー取得前にテナント所有権を検証**） | `<pkg>._lock_<aggregate>()` / `<pkg>._lock_tenant()` | XX003 |
| 5 | 対象の存在・状態（遷移許可） | `<pkg>._assert_transition()` | XX003 / XX004 |
| 6 | 立場・権限（管理者 / 担当者 / 名義人 / 委任） | `<pkg>._assert_<role>()` | XX005 / XX008 |
| 7 | 固有制約（重複・自己承認禁止など） | — | XX006 |
| 7' | リビジョン一致（楽観ロック対象のエンティティのみ。FOR UPDATE 取得後に比較） | — | XX010 |
| 8 | 本体処理 → 監査 `<pkg>._audit()` + イベント `<pkg>._event()`（状態遷移と同一 Tx）→ `<pkg>._idem_commit()` | — | — |

- 検査順は**原則**であり、例外（冪等再送判定が固有引数の検証より先に確定する、管理 SP で立場が存在より先、
  など）は許容するが、**ERRCODE の契約自体は不変**。例外は conventions.md に「検査順の注記」として列挙する
- 存在の有無を漏らさない: 他テナントの ID は「不在」（XX003）として扱い、「別テナントに存在する」と
  区別できる応答を返さない
- 読取 SP の検査は 1 → 2 → 閲覧権限（XX005）→ 本体

## 3. ERRCODE（5 文字 SQLSTATE、独自プレフィクス）

`RAISE EXCEPTION USING ERRCODE = 'XX00n', MESSAGE = '<英語の短文>', DETAIL = jsonb_build_object(...)::text`

| code | 意味（3 例で共通の骨格。題材で意味を具体化する） |
|---|---|
| XX001 | 入力不正（NULL・型・値域・必須属性欠落・冪等キー欠落） |
| XX002 | 主体またはテナントが無効（不在・別テナント・disabled・suspended） |
| XX003 | 対象不在（他テナントの ID を含む） |
| XX004 | 状態不正（遷移許可違反・終端後操作・版不一致） |
| XX005 | 立場違反（必要なロールがない・担当者でない・閲覧マトリクス違反） |
| XX006 | 制約違反（重複・自己承認禁止・期間重複・リビジョン不一致） |
| XX007 | 冪等キー競合（同一キー + 異なる入力） |
| XX008 | 委任・代理不正（不在・期間外・取消済み） |
| XX009 | 定義不正（公開時検証違反。DETAIL に違反一覧） |
| XX010 | リビジョン不一致（楽観ロック。呼出側は再読込して再提示 — §7 楽観ロック） |

- 呼出側が分岐してよい契約は **ERRCODE のみ**。MESSAGE / DETAIL は診断用で後方互換の対象外
- コードの追加は末尾追記のみ。意味の変更・再割当はしない（既存コードの意味を狭める場合は ADR）
- 新パッケージは XX001〜XX010 の骨格をそのまま採用し、不要な番号は「予約（未使用）」と書いて欠番にしない
  （型紙 workflow-component はリビジョン不一致を WF006 に相乗りさせているが、新パッケージは XX010 を使う —
  呼出側の対処が他の制約違反と異なるため。2026-09-25 発注者決定）

## 4. 冪等性

- 全外部書込 SP は**テナントスコープ + 実行者束縛**の冪等キー `p_idem_key`（必須）を取り、
  `<pkg>.idempotency_key` に入力ハッシュと結果を記録する
- 入力ハッシュ = `sha256(jsonb_build_object('actor', p_actor_id, ['on_behalf_of', ...,] 固有引数...)::text)`
  （`<pkg>._input_hash()`）。**監査注記（comment / decision_basis）はハッシュ対象外**:
  同一キーで注記だけ変えた再送は初回の結果を再返却する
- 同一キーの並行実行は `_idem_begin` 内の advisory lock（tenant × SP 名 × キー）で直列化する
- 適用除外（ステップ単位の再検証で自然に冪等なバッチ、削除 SP など）は conventions.md に**列挙**し ADR を添える
- 冪等キー記録は保持期間つき（クリーンアップ SP で削除。UPDATE は与えない）

## 5. SECURITY DEFINER とロール

| ロール | 性格 | 権限 |
|---|---|---|
| `<pkg>_owner` | オブジェクト所有。DEFINER 関数の実行主体 | NOLOGIN・NOINHERIT。全表 SELECT/INSERT/UPDATE/DELETE（追記専用表は除く） |
| `<pkg>_app` | アプリ接続 | 公開 SP の EXECUTE のみ。テーブルは SELECT も不可 |
| `<pkg>_agent` | AI サービスアカウント接続 | 代理系 SP のみ。**さらに実行者はサービスアカウント種別に限る**（`_assert_agent_actor`） |
| `<pkg>_batch` | バッチ接続（テナント横断） | 期限処理・クリーンアップ SP のみ |
| `<pkg>_readonly` | 運用・保守 | 全表 SELECT のみ。アプリには配らない |

- 公開 SP は `SECURITY DEFINER SET search_path = <pkg>, extensions, pg_temp`、所有者 `<pkg>_owner`
- 内部関数（`_` prefix）は INVOKER・EXECUTE 未配布（DEFINER 関数から owner 文脈で呼ばれる）
- **fail-closed**: grants は再適用のたびに **PUBLIC を含む** `REVOKE ALL` → 明示 `GRANT`。
  公開関数の分類（app / agent / batch）は 900_grants の配列に列挙し、**未分類の公開関数があれば
  RAISE で止める**（分類漏れを無言で残さない）
- 追記専用表（監査・投票・不変スナップショット）は `<pkg>_owner` からも UPDATE / DELETE / TRUNCATE を剥がす。
  seed 固定表（ステータスマスタ・遷移許可）は INSERT / UPDATE / DELETE を剥がす
- outbox・冪等キー表は追記 + 限定 UPDATE（例: `UPDATE (acked_at)` の列指定 GRANT）+ 保持期限切れ DELETE のみ
- 配布状態は**カタログからテスト**する（superuser で実行すると素通しになるため、GRANT の有無を
  `has_function_privilege` / `has_table_privilege` で検証する）

## 6. 監査・イベント

- 全書込 SP は末尾で `<pkg>._audit()`（action = SP 名、実行者と名義人の**表示名スナップショット**、
  operation_kind は SP が導出）→ 必要なら `<pkg>._event()`（outbox INSERT + `pg_notify('<pkg>_change', id のみ)`）
- outbox は**配送バッファであって監査証跡ではない**。保持期間 + クリーンアップ SP を持つ
- `pg_notify` の payload はイベント ID のみ（テナント情報を載せない — 越境防止）
- イベント語彙（event_type）は conventions.md に**正表**を置き、追加・変更は表の更新 + ADR を伴う
  （呼出側コンシューマの分岐対象なので契約）
- 時刻は `<pkg>.current_time()` ラッパ経由。実体は **clock_timestamp()（実時刻・VOLATILE）** で
  `now()`（Tx 開始時刻）ではない。長時間 Tx で失効判定・期限計算が古い時刻に固定されるのを防ぐ。
  テストではラッパを差し替えて時点ロジックを検証する

## 7. トランザクション・同時実行

- SP はトランザクション制御をしない（BEGIN / COMMIT は呼出側）
- **1 トランザクション = 1 書込 SP 呼出し**。同一 Tx で複数の書込 SP を呼ぶと、冪等キーの advisory
  lock（Tx 終了まで保持）と集約ロックの取得順序でデッドロックが成立し得る（外部レビュー指摘 → 規約化して受容）
- 1 トランザクション = 1 集約。一括操作はコアに持たない（単票 SP を独立 Tx で反復呼出し）
- **check-then-write を書いたら必ず write skew を疑う**。READ COMMITTED では「検査して書く」は並行実行で
  破れる。対策の既定形は集約単位の `pg_advisory_xact_lock` を**全書込 SP の冒頭で一律に取る**こと。
  細粒度ロックは先行読取が保護されず穴が残る。管理系パッケージの書込頻度なら直列化の代償は無視できる
- READ COMMITTED 前提と「1 Tx = 1 集約」は利用者契約（SKILL.md / README）に明記する

### 楽観ロック（revision） — 編集系エンティティのみ

DB 内の競合は上の悲観ロックで閉じている。楽観ロックが解くのは別の問題 — **「読む → 人間が数分編集する →
書く」という Tx をまたぐ作業での lost update** — なので、置く対象をそれに限定する。

- **対象**: 人間の編集セッションを挟んで内容を置き換えるエンティティ（DRAFT 定義・マスタ / 定義データ・
  自由記述欄）。**状態遷移系には置かない**（古い画面からの操作は遷移表が XX004 で止める。それが
  状態レベルの楽観チェックになっている）。対象エンティティは conventions.md §9 の欄に列挙する
- **列**: `revision int NOT NULL DEFAULT 0`。書込 SP が置換のたびに +1。`xmin`（無関係な UPDATE や周回で
  変わる）や `updated_at` の等値比較（時刻分解能依存）は使わない
- **引数**: `p_expected_revision int` を**必須**にする。NULL は XX001（NULL 比較は素通りする — 型紙で
  外部レビュー HIGH になった穴）
- **検査位置**: 検査順の 7'（集約の FOR UPDATE 取得後）。不一致は XX010、DETAIL に expected / current
- **冪等ハッシュに revision を含める**（同一キーで版だけ違う再送は別操作）
- **返却**: 書込結果 jsonb と詳細照会 SP の両方に新しい `revision` を載せる（呼出側が次の編集に使う）
- **編集方式**: 部分編集 API は作らず**丸ごと置換**（部分編集は版の意味が壊れやすい）
- `lock_no`（advisory lock のキー列）は楽観ロックではない。混同しない（sql-style.md 命名表）

## 8. マルチテナント境界

- ID を受け取る引数**すべて**に所属検証。検査（定義時）・適用（照会時）・一意制約の **3 点セット**で守る
  （1 箇所でも漏れると越境が成立する。レビューで実害まで再現済み）
- 全テーブルに `tenant_id`。`UNIQUE (tenant_id, id)` + 複合 FK でクロステナント参照を構造的に禁止。
  一意制約にもテナント列を含める
- 他パッケージの主体（iam の principal など）への参照は **soft reference**（FK を張らない）。
  書込時に提供側の公開 API で検証し、表示名はスナップショットで持つ

## 9. パッケージ固有に埋める欄（conventions.md にコピー後、必ず書く）

- [ ] `<pkg>` / `XX` / ロール名の置換
- [ ] ERRCODE 表の各コードに題材固有の意味を列挙（例: XX006 = 二重投票・自己承認禁止・…）
- [ ] 検査順の例外（注記）
- [ ] 冪等の適用除外 SP と ADR 番号
- [ ] ロール別 EXECUTE 配布の関数一覧（900_grants の配列と一致させる）
- [ ] 追記専用表・seed 固定表の一覧
- [ ] イベント語彙の正表
- [ ] 集約の定義（何を 1 Tx の単位にするか）とロック関数の名前
- [ ] 楽観ロック（revision）を置くエンティティの一覧と、対応する置換 SP（無ければ「該当なし」と書く）
