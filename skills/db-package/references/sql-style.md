# SQL / PL/pgSQL コーディング規約 — SP を書く直前とレビュー時に読む

sp-conventions.md が「SP の外形（引数・検査順・ERRCODE・権限）」を決めるのに対し、この文書は
**コードの書き方**を決める。型紙（workflow-component）の db/ をそのまま一般化したもの。
レビューでは本文書の各項目を機械的にチェックできる形にしてある。

## 1. ファイル構成

```
db/
  schema/      V{番号3桁}__{snake_name}.sql   # マイグレーション。適用済みは不変
  procedures/  {番号3桁}_{snake_name}.sql     # 1 SP = 1 ファイル。CREATE OR REPLACE で冪等再適用
  tests/       {番号3桁}_{snake_name}.sql     # pgTAP
```

- procedures の番号帯: `000` 共通内部関数 / `010` 集約コア / `1xx` 実行系 / `2xx` 定義・管理系 /
  `3xx` 権限系 / `4xx` 運用救済系 / `5xx` 照会系 / `900` grants（**末尾で権限を確定**）
- V ファイルは番号順に psql ランナーで適用。**適用済みファイルは変更しない**。変更は新 V ファイル
- 各ファイルの 1 行目はヘッダコメント: SP-ID / 対応 UC / 主要 ADR（例: `-- SP-002 cast_vote（UC-002 …。ADR-002）`）
- V ファイルの冒頭コメントに「なぜ」を書く（ADR 番号 + 1〜3 行）。テーブル・列には `COMMENT ON` で
  エンティティ ID（E-xxx）と要点を残す（ERD をカタログから自動生成するため）

## 2. 命名

| 対象 | 規則 | 例 |
|---|---|---|
| スキーマ | 短い小文字 1 語 | `wf`, `iam` |
| 公開 SP | `動詞_目的語`。UC と 1:1 | `cast_vote`, `list_my_tasks`, `get_request_detail` |
| 内部関数 | `_` prefix。EXECUTE は配布しない | `_principal`, `_idem_begin`, `_assert_admin` |
| 集約コア | `_<sp>_core`（公開 SP と代理 SP が共用する本体） | `_cast_vote_core` |
| 引数 | `p_` prefix | `p_tenant_id`, `p_idem_key` |
| ローカル変数 | `v_` prefix。record 型のループ変数は 1 文字可 | `v_display`, `v_row`, `FOR r IN ...` |
| 定数配列 | `c_` prefix | `c_app text[]` |
| テーブル | 単数形 snake_case | `request_step`, `status_transition` |
| 列 | snake_case。時刻は `_at`、主体は `_by`、表示名スナップショットは `_display` | `occurred_at`, `acted_by`, `acted_by_display` |
| 状態列 | `status text` + `-- domain: <名>` コメント。ENUM は使わない | `status text NOT NULL DEFAULT 'DRAFT', -- domain: route_version` |
| 状態値 | 大文字 SNAKE | `ACTIVE`, `FORCE_CLOSED` |
| ロール | `<pkg>_owner` / `_app` / `_agent` / `_batch` / `_readonly` | `wf_app` |
| advisory lock キー | `'<pkg>_<対象>:' || id` を `hashtextextended(…, 0)` | `'wf_request:' \|\| lock_no` |
| advisory lock キー列 | `lock_no bigint GENERATED ALWAYS AS IDENTITY, UNIQUE (lock_no)`。**楽観ロックではない**（整数ハッシュ用） | `request.lock_no` |
| 楽観ロック版 | `revision int NOT NULL DEFAULT 0`。編集系エンティティのみ。引数は `p_expected_revision` | `route_version.revision` |

## 3. 関数の定型

```sql
-- SP-nnn <name>（UC-nnn <要約>。ADR-nnn）
CREATE OR REPLACE FUNCTION <pkg>.<name>(
    p_tenant_id uuid, p_actor_id uuid, <必須固有引数>, p_idem_key text,
    <任意固有引数> DEFAULT NULL
) RETURNS jsonb LANGUAGE plpgsql
SECURITY DEFINER SET search_path = <pkg>, extensions, pg_temp AS $$
DECLARE v_display text; v_hash text; v_stored jsonb; v_result jsonb;
BEGIN
    v_display := <pkg>._principal(p_tenant_id, p_actor_id);
    v_hash := <pkg>._input_hash(jsonb_build_object('actor', p_actor_id, <固有引数のキー/値...>));
    v_stored := <pkg>._idem_begin(p_tenant_id, '<name>', p_idem_key, v_hash);
    IF v_stored IS NOT NULL THEN RETURN v_stored; END IF;

    v_result := <pkg>._<name>_core(p_tenant_id, p_actor_id, v_display, ...);

    PERFORM <pkg>._idem_commit(p_tenant_id, '<name>', p_idem_key, v_hash, v_result);
    RETURN v_result;
END $$;
```

- 書込 SP の戻り値は `jsonb`（冪等再返却のため保存可能な形に統一）。読取 SP は `RETURNS TABLE (...)`。
  **名前付き複合型は作らない**（変更管理が効かない）
- `LANGUAGE sql` は単純な INSERT / SELECT ラッパのみ。分岐・例外があれば plpgsql
- volatility を必ず明示: 読取・検証は `STABLE`、純関数は `IMMUTABLE`、時刻ラッパは **`VOLATILE`**
  （clock_timestamp を包むため。STABLE にすると Tx 内で固定される）
- 代理 SP は本体を `_<name>_core` に置き、公開 SP・代理 SP の両方がそれを呼ぶ（検査順の重複実装を防ぐ）
- オブジェクト参照は**完全修飾**（`<pkg>.table`）。search_path に依存しない
- 動的 SQL は原則書かない。必要なら `format()` の `%I` / `%L` 経由のみ（grants の一括処理など）。
  DEFINER 関数内で文字列連結の EXECUTE を書いたらレビューで差し戻す
- `SELECT ... INTO` の後は `IF NOT FOUND` か `IS NULL` で必ず不在を処理する。`STRICT` は使わない
  （例外の SQLSTATE が P0002 になり ERRCODE 契約から外れる）
- 集約の行ロックは `SELECT ... FOR UPDATE`。ただし**先に advisory lock**（§7 sp-conventions）

## 4. 例外の書式

```sql
RAISE EXCEPTION USING ERRCODE = 'XX004', MESSAGE = 'transition not allowed',
    DETAIL = jsonb_build_object('domain', p_domain, 'from', p_from, 'to', p_to)::text;
```

- MESSAGE は英語の短文・固定文字列（値を埋め込まない。値は DETAIL へ）
- DETAIL は `jsonb_build_object(...)::text`。診断に必要なキーだけ。**他テナントの存在を示唆する情報は載せない**
- `RAISE EXCEPTION '...'`（ERRCODE なし = P0001）は禁止。**唯一の例外**は 900_grants の
  「未分類の公開関数」検出など、実行時でなく適用時に止めるための RAISE
- `EXCEPTION WHEN others` で握りつぶさない。型変換の検証など限定用途で使う場合は、必ず XX001 に
  読み替えて再 RAISE する

## 5. スキーマ（V ファイル）

- 全テーブル: `id uuid NOT NULL DEFAULT gen_random_uuid()`, `tenant_id uuid NOT NULL`,
  `PRIMARY KEY (id)`, **`UNIQUE (tenant_id, id)`**。子テーブルは `FOREIGN KEY (tenant_id, parent_id)
  REFERENCES parent (tenant_id, id)` の複合 FK でクロステナント参照を構造的に禁止
- 一意制約は必ず `tenant_id` を含む（`UNIQUE (tenant_id, code)`）
- 区分値は `text` + CHECK またはドメイン型。**ENUM は使わない**（削除・リネーム不可。値変更は
  `ALTER DOMAIN` でトランザクショナルに）
- 状態は ENUM でなく**ステータスマスタ + 遷移許可テーブル**（seed 固定。status 列から master への FK は
  張らず、SP 内で `_assert_transition` で検証）
- 他パッケージへの参照列は FK を張らず、`-- soft reference（<提供側>。書込時に <API> で検証）` とコメント
- 期間重複は `EXCLUDE USING gist` + `btree_gist`（`extensions` スキーマに置く）
- 集約テーブルには `lock_no bigint GENERATED ALWAYS AS IDENTITY` + `UNIQUE (lock_no)`（advisory lock 用）。
  楽観ロック対象（sp-conventions §7）の編集系テーブルにだけ `revision int NOT NULL DEFAULT 0` を足す
- 拡張は `CREATE EXTENSION IF NOT EXISTS ... SCHEMA extensions`。`public` に置かない
- 列の追加は `ALTER TABLE ... ADD COLUMN` のみ。列の削除・型変更・リネームはしない（新列 + 移行 ADR）
- **関数のシグネチャ変更（引数追加）は同じ V ファイル内で旧シグネチャを `DROP FUNCTION`** してから
  再作成する。`CREATE OR REPLACE` は別オーバーロードを追加するだけで置換しない（checklist.md
  「引数追加時の注意」）。同 V 内で grants 再適用まで行う

## 6. grants（900_grants.sql の骨格）

1. ロール作成は `DO $$ BEGIN CREATE ROLE ...; EXCEPTION WHEN duplicate_object THEN NULL; END $$`（冪等）
2. `GRANT USAGE ON SCHEMA` を必要なロールへ
3. **`REVOKE ALL ON ALL TABLES / SEQUENCES / FUNCTIONS IN SCHEMA <pkg> FROM PUBLIC, <接続ロール全部>`**
4. owner へテーブル権限、追記専用・seed 固定・outbox の REVOKE（sp-conventions §5）
5. 公開関数を `c_app` / `c_agent` / `c_batch` の配列で分類し、`pg_proc` をループして
   `ALTER FUNCTION ... OWNER TO <pkg>_owner` → 公開なら `SECURITY DEFINER SET search_path` と GRANT、
   `_` prefix でも分類配列にも無い関数は `RAISE EXCEPTION 'unclassified public function %'`
6. `ALTER DEFAULT PRIVILEGES` は使わない（IN SCHEMA 形は組込み既定を打ち消せず、グローバル形は
   他パッケージを巻き込む）。新規関数の既定 EXECUTE(PUBLIC) はマイグレーションランナーの
   後処理で 900_grants を再実行して剥がす

## 7. テスト（pgTAP）

- `SET search_path TO tap, <pkg>, extensions, public;` → `BEGIN; SELECT plan(N); ... SELECT finish(); ROLLBACK;`
  で各ファイル独立
- fixture は**公開 SP で毎回生成**。固定 UUID を直書きしてよいのは構造制約テスト（V ファイル検証）の
  直接 INSERT のみで、ファイル冒頭に「SP を介さない直接 INSERT はこのテスト専用」と明記
- SP ごとに単体（正常系 + **全 ERRCODE**）: `throws_ok(sql, 'XX004', NULL, '説明')`。ERRCODE を
  文字列比較し、MESSAGE には依存しない
- **弱いアサーション禁止**: `ok(true)`、`lives_ok` だけ、件数だけの比較はレビューで差し戻す。
  結果値・状態値まで `is()` / `results_eq()` で比較する
- 権限マトリクスは `has_function_privilege('<pkg>_app', '<pkg>.<fn>(uuid, uuid, ...)', 'EXECUTE')` と
  `has_table_privilege` で**カタログから**検証（superuser で実行すると素通しになるため）
- 時点ロジックは `current_time()` ラッパを差し替えて検証（テスト内で `CREATE OR REPLACE FUNCTION
  <pkg>.current_time() ... AS $$ SELECT '2026-01-01'::timestamptz $$` → ROLLBACK で戻る）
- 並行実行は 2 セッション bash（`psql` × 2 + `sleep`）。二重実行・競合・冪等キー競合を最低限含める
- traceability の T 列（UC ⇄ SP ⇄ テスト）を埋めて全件 green

## 8. レビューチェックリスト（SP 1 本ごと）

- [ ] ヘッダコメントに SP-ID / UC / ADR
- [ ] 引数順が sp-conventions §1 に一致。`p_idem_key` の位置
- [ ] `SECURITY DEFINER SET search_path = <pkg>, extensions, pg_temp`（900_grants でも再適用）
- [ ] 検査順 §2 の各段が揃っている（省略はその SP の仕様書に理由）
- [ ] 全 RAISE に ERRCODE。MESSAGE 固定文字列、DETAIL に json
- [ ] 状態更新はヘルパ経由で遷移表を通る（`UPDATE ... SET status =` を直接書いていない）
- [ ] 楽観ロック対象なら `p_expected_revision` 必須（NULL → XX001）・FOR UPDATE 後に比較・不一致 XX010・
      `revision + 1` で更新・結果に新 revision・冪等ハッシュに revision
- [ ] 末尾で `_audit`（+ `_event`）→ `_idem_commit`
- [ ] 完全修飾・動的 SQL なし・`STRICT` なし・`EXCEPTION WHEN others` なし
- [ ] 900_grants の分類配列に登録済み
- [ ] テストに正常系 + 全 ERRCODE + 結果値比較
