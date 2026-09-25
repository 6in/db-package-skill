# 共通 SQL の雛形（000_common.sql / 900_grants.sql） — 作業 #0 の vendoring 時に読む

型紙（workflow-component の db/procedures/000_common.sql と 900_grants.sql）から、題材に依存しない
部分だけを `<pkg>` / `XX` プレースホルダで抜き出したもの。型紙リポジトリが手元にあればそちらを
生読みして写してよい。無ければこの雛形から起こす。**規約の出典は sp-conventions.md と
sql-style.md** で、この文書はその実装形にすぎない。

各ブロックは独立にコピーできる。順序は 000_common.sql 内の並びと同じ。

## 1. 時刻ラッパ（VOLATILE・clock_timestamp）

```sql
-- 時刻。テストで差し替え可能にするラッパ。now()（Tx 開始時刻）ではなく clock_timestamp()（実時刻）
CREATE OR REPLACE FUNCTION <pkg>.current_time() RETURNS timestamptz
LANGUAGE sql VOLATILE AS $$ SELECT clock_timestamp() $$;
```

テストでの差し替え例（ROLLBACK で元に戻る）:

```sql
CREATE OR REPLACE FUNCTION <pkg>.current_time() RETURNS timestamptz
LANGUAGE sql VOLATILE AS $$ SELECT '2026-01-01 00:00:00+09'::timestamptz $$;
```

## 2. 遷移許可の検証

```sql
CREATE OR REPLACE FUNCTION <pkg>._assert_transition(p_domain text, p_from text, p_to text)
RETURNS void LANGUAGE plpgsql STABLE AS $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM <pkg>.status_transition
                    WHERE domain = p_domain AND from_code = p_from AND to_code = p_to) THEN
        RAISE EXCEPTION USING ERRCODE = 'XX004', MESSAGE = 'transition not allowed',
            DETAIL = jsonb_build_object('domain', p_domain, 'from', p_from, 'to', p_to)::text;
    END IF;
END $$;
```

対応するテーブル（V001）:

```sql
CREATE TABLE <pkg>.status_master (
    domain text NOT NULL, code text NOT NULL, label text NOT NULL,
    PRIMARY KEY (domain, code)
);
CREATE TABLE <pkg>.status_transition (
    domain text NOT NULL, from_code text NOT NULL, to_code text NOT NULL,
    PRIMARY KEY (domain, from_code, to_code),
    FOREIGN KEY (domain, from_code) REFERENCES <pkg>.status_master (domain, code),
    FOREIGN KEY (domain, to_code)   REFERENCES <pkg>.status_master (domain, code)
);
```

状態更新はヘルパに閉じ込める（全変更経路が遷移表を通ることをレビュー対象にする）:

```sql
CREATE OR REPLACE FUNCTION <pkg>._set_<aggregate>_status(p_tenant_id uuid, p_id uuid, p_to text)
RETURNS void LANGUAGE plpgsql AS $$
DECLARE v_from text;
BEGIN
    SELECT status INTO v_from FROM <pkg>.<aggregate> WHERE tenant_id = p_tenant_id AND id = p_id FOR UPDATE;
    IF v_from IS NULL THEN
        RAISE EXCEPTION USING ERRCODE = 'XX003', MESSAGE = '<aggregate> not found',
            DETAIL = jsonb_build_object('id', p_id)::text;
    END IF;
    PERFORM <pkg>._assert_transition('<aggregate>', v_from, p_to);
    UPDATE <pkg>.<aggregate> SET status = p_to, updated_at = <pkg>.current_time()
     WHERE tenant_id = p_tenant_id AND id = p_id;
END $$;
```

## 3. 主体の有効性検証 + 表示名スナップショット（他パッケージ契約の呼び出し）

```sql
-- 提供側（iam）の公開 API だけを使う。生テーブル参照はしない（soft reference）
CREATE OR REPLACE FUNCTION <pkg>._principal(p_tenant_id uuid, p_principal_id uuid)
RETURNS text LANGUAGE plpgsql STABLE AS $$
DECLARE v_display text; v_status text; v_tenant_status text;
BEGIN
    IF p_tenant_id IS NULL OR p_principal_id IS NULL THEN
        RAISE EXCEPTION USING ERRCODE = 'XX001', MESSAGE = 'tenant_id / principal_id is required';
    END IF;
    SELECT s.display_name, s.status, s.tenant_status
      INTO v_display, v_status, v_tenant_status
      FROM iam.principal_snapshot(p_tenant_id, p_principal_id) s;
    IF v_display IS NULL OR v_status <> 'active' OR v_tenant_status <> 'active' THEN
        RAISE EXCEPTION USING ERRCODE = 'XX002',
            MESSAGE = 'principal not active or not in tenant',
            DETAIL = jsonb_build_object('tenant_id', p_tenant_id, 'principal_id', p_principal_id,
                                        'status', v_status, 'tenant_status', v_tenant_status)::text;
    END IF;
    RETURN v_display;
END $$;
```

iam を使わないパッケージでは、自前の主体表に対して同じ形（有効性 + テナント所属 + 表示名）で書く。

## 4. 2 層ロック（テナント単位 / 集約単位）

```sql
-- テナント単位（定義・ロール・委任など集約を持たない書込）
CREATE OR REPLACE FUNCTION <pkg>._lock_tenant(p_tenant_id uuid) RETURNS void
LANGUAGE sql AS $$
    SELECT pg_advisory_xact_lock(hashtextextended('<pkg>_tenant:' || p_tenant_id::text, 0))
$$;

-- 集約単位。キー取得前にテナント所有権を検証（XX003）し、ロック後に行を返す
CREATE OR REPLACE FUNCTION <pkg>._lock_<aggregate>(p_tenant_id uuid, p_id uuid)
RETURNS <pkg>.<aggregate> LANGUAGE plpgsql AS $$
DECLARE v_lock_no bigint; v_row <pkg>.<aggregate>;
BEGIN
    SELECT lock_no INTO v_lock_no FROM <pkg>.<aggregate>
     WHERE tenant_id = p_tenant_id AND id = p_id;
    IF v_lock_no IS NULL THEN
        RAISE EXCEPTION USING ERRCODE = 'XX003', MESSAGE = '<aggregate> not found',
            DETAIL = jsonb_build_object('id', p_id)::text;
    END IF;
    PERFORM pg_advisory_xact_lock(hashtextextended('<pkg>_<aggregate>:' || v_lock_no::text, 0));
    SELECT * INTO v_row FROM <pkg>.<aggregate>
     WHERE tenant_id = p_tenant_id AND id = p_id FOR UPDATE;
    RETURN v_row;
END $$;
```

- `lock_no bigint GENERATED ALWAYS AS IDENTITY` を集約テーブルに持たせ、uuid でなく整数でハッシュする
- ロックの取得順序は**必ず「冪等キー → テナント or 集約」**。逆順の SP が 1 本でもあるとデッドロック

## 5. 冪等キー（_input_hash / _idem_begin / _idem_commit）

```sql
CREATE OR REPLACE FUNCTION <pkg>._input_hash(p_args jsonb) RETURNS text
LANGUAGE sql IMMUTABLE AS $$
    SELECT encode(extensions.digest(p_args::text, 'sha256'), 'hex')
$$;

-- 戻り値: 保存済みの結果（再返却用）。NULL = 初回実行（続行してよい）
CREATE OR REPLACE FUNCTION <pkg>._idem_begin(
    p_tenant_id uuid, p_sp_name text, p_idem_key text, p_input_hash text
) RETURNS jsonb LANGUAGE plpgsql AS $$
DECLARE v_row <pkg>.idempotency_key;
BEGIN
    IF p_idem_key IS NULL OR p_idem_key = '' THEN
        RAISE EXCEPTION USING ERRCODE = 'XX001', MESSAGE = 'idempotency key is required',
            DETAIL = jsonb_build_object('sp', p_sp_name)::text;
    END IF;
    PERFORM pg_advisory_xact_lock(hashtextextended(
        '<pkg>_idem:' || p_tenant_id::text || ':' || p_sp_name || ':' || p_idem_key, 0));
    SELECT * INTO v_row FROM <pkg>.idempotency_key
     WHERE tenant_id = p_tenant_id AND sp_name = p_sp_name AND idem_key = p_idem_key;
    IF NOT FOUND THEN
        RETURN NULL;
    END IF;
    IF v_row.input_hash <> p_input_hash THEN
        RAISE EXCEPTION USING ERRCODE = 'XX007', MESSAGE = 'idempotency key conflict',
            DETAIL = jsonb_build_object('sp', p_sp_name, 'idem_key', p_idem_key)::text;
    END IF;
    RETURN coalesce(v_row.result, '{}'::jsonb);
END $$;

CREATE OR REPLACE FUNCTION <pkg>._idem_commit(
    p_tenant_id uuid, p_sp_name text, p_idem_key text, p_input_hash text, p_result jsonb
) RETURNS void LANGUAGE sql AS $$
    INSERT INTO <pkg>.idempotency_key (tenant_id, sp_name, idem_key, input_hash, result)
    VALUES (p_tenant_id, p_sp_name, p_idem_key, p_input_hash, p_result)
$$;
```

対応するテーブル:

```sql
CREATE TABLE <pkg>.idempotency_key (
    tenant_id  uuid NOT NULL,
    sp_name    text NOT NULL,
    idem_key   text NOT NULL,
    input_hash text NOT NULL,
    result     jsonb,
    created_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, sp_name, idem_key)
);
```

- `extensions.digest` は pgcrypto（`CREATE EXTENSION IF NOT EXISTS pgcrypto SCHEMA extensions`）
- 呼出側でのハッシュ対象: `jsonb_build_object('actor', p_actor_id, [ 'on_behalf_of', ..., ] 固有引数...)`。
  **comment / decision_basis は含めない**

## 6. 監査 + イベント outbox

```sql
CREATE OR REPLACE FUNCTION <pkg>._audit(
    p_tenant_id uuid, p_action text, p_target_kind text, p_target_id uuid,
    p_acted_by uuid, p_acted_by_display text,
    p_on_behalf_of uuid, p_on_behalf_of_display text, p_operation_kind text,
    <題材固有の任意列> DEFAULT NULL ...
) RETURNS void LANGUAGE sql AS $$
    INSERT INTO <pkg>.<audit_table> (
        tenant_id, occurred_at, action, target_kind, target_id,
        acted_by, acted_by_display, on_behalf_of, on_behalf_of_display, operation_kind, ...)
    VALUES (p_tenant_id, <pkg>.current_time(), p_action, p_target_kind, p_target_id,
            p_acted_by, p_acted_by_display, p_on_behalf_of, p_on_behalf_of_display, p_operation_kind, ...)
$$;

CREATE OR REPLACE FUNCTION <pkg>._event(
    p_tenant_id uuid, p_event_type text, p_target_kind text, p_target_id uuid
) RETURNS void LANGUAGE plpgsql AS $$
DECLARE v_id bigint;
BEGIN
    INSERT INTO <pkg>.event_outbox (tenant_id, event_type, target_kind, target_id, occurred_at)
    VALUES (p_tenant_id, p_event_type, p_target_kind, p_target_id, <pkg>.current_time())
    RETURNING id INTO v_id;
    -- ウェイクアップ専用。payload はイベント ID のみ（テナント情報の越境防止）
    PERFORM pg_notify('<pkg>_change', v_id::text);
END $$;
```

outbox テーブルは `id bigint GENERATED ALWAYS AS IDENTITY`, `acked_at timestamptz`, `occurred_at` を持ち、
`fetch_events` / `ack_events`（app 向け公開 SP）と `cleanup_expired`（batch 向け、保持期限切れ DELETE）を対にする。

## 7. 立場の判定

```sql
CREATE OR REPLACE FUNCTION <pkg>._has_role(p_tenant_id uuid, p_principal_id uuid, p_role_code text)
RETURNS boolean LANGUAGE sql STABLE AS $$
    SELECT EXISTS (
        SELECT 1 FROM <pkg>.<role_assignment> a
          JOIN <pkg>.<role> r ON r.tenant_id = a.tenant_id AND r.id = a.role_id
         WHERE a.tenant_id = p_tenant_id AND a.principal_id = p_principal_id AND r.code = p_role_code)
$$;

CREATE OR REPLACE FUNCTION <pkg>._assert_admin(p_tenant_id uuid, p_principal_id uuid)
RETURNS void LANGUAGE plpgsql STABLE AS $$
BEGIN
    IF NOT <pkg>._has_role(p_tenant_id, p_principal_id, '<pkg>_admin') THEN
        RAISE EXCEPTION USING ERRCODE = 'XX005', MESSAGE = 'admin role required',
            DETAIL = jsonb_build_object('principal_id', p_principal_id)::text;
    END IF;
END $$;
```

## 8. AI エージェント接続の実行者検証（代理系 SP を持つ場合）

```sql
-- <pkg>_agent 接続の実行者はサービスアカウント種別に限る。人間 principal を p_actor_id に渡して
-- その人物の委任で別名義を得る「人間へのなりすまし」を DB 側でも遮断する。
-- pg_has_role は superuser で常に真になるため、pg_auth_members の直接メンバーシップを見る
CREATE OR REPLACE FUNCTION <pkg>._assert_agent_actor(p_tenant_id uuid, p_actor_id uuid)
RETURNS void LANGUAGE plpgsql STABLE AS $$
DECLARE v_kind text;
BEGIN
    IF EXISTS (SELECT 1 FROM pg_auth_members m
                 JOIN pg_roles r ON r.oid = m.roleid
                 JOIN pg_roles u ON u.oid = m.member
                WHERE r.rolname = '<pkg>_agent' AND u.rolname = session_user) THEN
        SELECT s.kind INTO v_kind FROM iam.principal_snapshot(p_tenant_id, p_actor_id) s;
        IF v_kind IS DISTINCT FROM 'service' THEN
            RAISE EXCEPTION USING ERRCODE = 'XX005',
                MESSAGE = '<pkg>_agent connections must act as a service account principal',
                DETAIL = jsonb_build_object('actor_id', p_actor_id)::text;
        END IF;
    END IF;
END $$;

-- 判断根拠は代理経由の全操作で必須
CREATE OR REPLACE FUNCTION <pkg>._require_basis(p_decision_basis text) RETURNS void
LANGUAGE plpgsql IMMUTABLE AS $$
BEGIN
    IF p_decision_basis IS NULL OR btrim(p_decision_basis) = '' THEN
        RAISE EXCEPTION USING ERRCODE = 'XX001', MESSAGE = 'decision_basis is required for delegated operations';
    END IF;
END $$;
```

## 9. 900_grants.sql の骨格（fail-closed）

```sql
DO $$
BEGIN
    BEGIN CREATE ROLE <pkg>_owner NOLOGIN NOINHERIT; EXCEPTION WHEN duplicate_object THEN NULL; END;
    BEGIN CREATE ROLE <pkg>_app      NOLOGIN; EXCEPTION WHEN duplicate_object THEN NULL; END;
    BEGIN CREATE ROLE <pkg>_agent    NOLOGIN; EXCEPTION WHEN duplicate_object THEN NULL; END;
    BEGIN CREATE ROLE <pkg>_batch    NOLOGIN; EXCEPTION WHEN duplicate_object THEN NULL; END;
    BEGIN CREATE ROLE <pkg>_readonly NOLOGIN; EXCEPTION WHEN duplicate_object THEN NULL; END;
END $$;

GRANT USAGE ON SCHEMA <pkg>      TO <pkg>_owner, <pkg>_app, <pkg>_agent, <pkg>_batch, <pkg>_readonly;
GRANT USAGE ON SCHEMA extensions TO <pkg>_owner, <pkg>_app, <pkg>_agent, <pkg>_batch;

-- fail-closed: PUBLIC を含む全接続ロールの既存権限を剥がしてから明示的に配り直す
REVOKE ALL ON ALL TABLES    IN SCHEMA <pkg> FROM PUBLIC, <pkg>_app, <pkg>_agent, <pkg>_batch, <pkg>_readonly;
REVOKE ALL ON ALL SEQUENCES IN SCHEMA <pkg> FROM PUBLIC, <pkg>_app, <pkg>_agent, <pkg>_batch, <pkg>_readonly;
REVOKE ALL ON ALL FUNCTIONS IN SCHEMA <pkg> FROM PUBLIC, <pkg>_app, <pkg>_agent, <pkg>_batch, <pkg>_readonly;

-- 連携先パッケージの公開 API（least privilege。提供側の app ロールごと付与はしない）
GRANT USAGE ON SCHEMA iam TO <pkg>_owner;
GRANT EXECUTE ON FUNCTION iam.principal_snapshot(uuid, uuid) TO <pkg>_owner;

-- owner: DEFINER 関数の実行主体
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA <pkg> TO <pkg>_owner;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA <pkg> TO <pkg>_owner;
-- 追記専用（所有者経路も塞ぐ）
REVOKE UPDATE, DELETE, TRUNCATE ON <pkg>.<audit_table>, <pkg>.<immutable_snapshot_table> FROM <pkg>_owner;
-- outbox / 冪等キー: 追記 + 限定 UPDATE + 保持期限切れ DELETE のみ
REVOKE UPDATE, TRUNCATE ON <pkg>.event_outbox FROM <pkg>_owner;
GRANT  UPDATE (acked_at) ON <pkg>.event_outbox TO <pkg>_owner;
REVOKE UPDATE, TRUNCATE ON <pkg>.idempotency_key FROM <pkg>_owner;
-- seed 固定表
REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON <pkg>.status_master, <pkg>.status_transition FROM <pkg>_owner;

GRANT SELECT ON ALL TABLES IN SCHEMA <pkg> TO <pkg>_readonly;

REVOKE ALL ON ALL FUNCTIONS IN SCHEMA <pkg> FROM PUBLIC;

DO $$
DECLARE
    c_agent text[] := ARRAY['<代理系 SP...>'];
    c_batch text[] := ARRAY['<期限処理>', 'cleanup_expired'];
    c_app   text[] := ARRAY['<通常の公開 SP...>', 'fetch_events', 'ack_events'];
    r record;
BEGIN
    FOR r IN SELECT p.oid::regprocedure AS sig, p.proname
               FROM pg_proc p JOIN pg_namespace n ON n.oid = p.pronamespace
              WHERE n.nspname = '<pkg>'
    LOOP
        EXECUTE format('ALTER FUNCTION %s OWNER TO <pkg>_owner', r.sig);
        IF r.proname = ANY (c_app || c_agent || c_batch) THEN
            EXECUTE format('ALTER FUNCTION %s SECURITY DEFINER SET search_path = <pkg>, extensions, pg_temp', r.sig);
            IF r.proname = ANY (c_batch) THEN
                EXECUTE format('GRANT EXECUTE ON FUNCTION %s TO <pkg>_batch', r.sig);
            ELSIF r.proname = ANY (c_agent) THEN
                EXECUTE format('GRANT EXECUTE ON FUNCTION %s TO <pkg>_agent, <pkg>_app', r.sig);
            ELSE
                EXECUTE format('GRANT EXECUTE ON FUNCTION %s TO <pkg>_app', r.sig);
            END IF;
        ELSIF left(r.proname, 1) <> '_' AND r.proname <> 'current_time' THEN
            -- 分類漏れの公開関数を無言で残さない（fail-closed）
            RAISE EXCEPTION '900_grants.sql: unclassified public function %', r.sig;
        END IF;
        -- 内部関数（_ prefix / current_time）は INVOKER のまま・EXECUTE 未配布
    END LOOP;
END $$;
```

配布状態のテスト（030_api_surface 相当）は、`c_app` 等の一覧と `has_function_privilege` の結果を
**カタログから**突き合わせ、`<pkg>_app` がテーブルに一切アクセスできないことも `has_table_privilege` で固定する。

## 10. 楽観ロック（revision）の検査 — 編集系エンティティの置換 SP にだけ書く

```sql
-- 列（V001）: revision int NOT NULL DEFAULT 0   -- 編集の世代（lost update 防止。sp-conventions §7）

-- SP 本体（検査順 1 → 2 → 3 → 4 → 5 → 7' の順。抜粋）
    IF p_expected_revision IS NULL THEN
        -- NULL は比較結果が NULL になり lost update 防止を素通りする
        RAISE EXCEPTION USING ERRCODE = 'XX001', MESSAGE = 'expected_revision is required';
    END IF;
    v_hash := <pkg>._input_hash(jsonb_build_object('actor', p_actor_id, 'id', p_id,
                                                   'def', p_definition, 'rev', p_expected_revision));
    ...
    SELECT * INTO v_row FROM <pkg>.<editable> WHERE tenant_id = p_tenant_id AND id = p_id FOR UPDATE;
    IF v_row.id IS NULL THEN
        RAISE EXCEPTION USING ERRCODE = 'XX003', MESSAGE = '<editable> not found';
    END IF;
    IF v_row.status <> 'DRAFT' THEN
        RAISE EXCEPTION USING ERRCODE = 'XX004', MESSAGE = 'only DRAFT can be edited',
            DETAIL = jsonb_build_object('status', v_row.status)::text;
    END IF;
    IF v_row.revision <> p_expected_revision THEN
        RAISE EXCEPTION USING ERRCODE = 'XX010', MESSAGE = 'revision mismatch (lost update)',
            DETAIL = jsonb_build_object('expected', p_expected_revision, 'current', v_row.revision)::text;
    END IF;

    -- 丸ごと置換（部分編集 API は作らない）
    ...
    UPDATE <pkg>.<editable>
       SET revision = revision + 1, updated_at = <pkg>.current_time()
     WHERE tenant_id = p_tenant_id AND id = p_id
    RETURNING revision INTO v_new_rev;

    v_result := jsonb_build_object('id', p_id, 'revision', v_new_rev);
```

- 詳細照会 SP の RETURNS TABLE にも `revision int` を含める（呼出側が次の置換で渡す）
- テスト: 正しい revision で成功 → 返却 revision が +1 / 古い revision で XX010 / NULL で XX001 /
  同一キー + 別 revision の再送は XX007（冪等キー競合）になることを固定する
