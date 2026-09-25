# ADR-001 楽観ロックの既定 — revision 列を編集系エンティティにだけ置く

- 日付: 2026-09-25（発注者決定、AskUserQuestion による選択式）
- 状態: 採択
- 反映先: SKILL.md「設計の既定値」/ references/sp-conventions.md §3・§2・§7・§9 /
  references/sql-style.md 命名表・§5・§8 / references/common-sql-patterns.md §10

## 背景

型紙 workflow-component では `lock_no` が advisory lock のハッシュキー（IDENTITY 整数）であり、
楽観ロックは route_version の `revision` + SP-103 の `p_expected_revision`（WF006 相乗り）の
1 箇所だけだった。新パッケージで「専用列を設けるか」が未決だった。

## 決定

1. **対象は編集系エンティティのみ**（DRAFT 定義・マスタ / 定義データ・自由記述）。状態遷移系には
   置かない。DB 内の競合は advisory lock + FOR UPDATE + 遷移表で閉じており、楽観ロックが解くのは
   Tx をまたぐ人間の編集セッションの lost update だけであるため
2. **列は `revision int NOT NULL DEFAULT 0`**。`xmin` / `updated_at` 比較は採らない。
   `p_expected_revision` は必須（NULL → XX001）、FOR UPDATE 後に比較、冪等ハッシュに含める、
   結果と詳細照会に新 revision を返す、丸ごと置換方式
3. **ERRCODE は専用の XX010**。呼出側の対処（再読込して再提示）が他の制約違反と異なるため。
   型紙の WF006 相乗りとは差が出るが、型紙側は変更しない
4. **`lock_no` の名前は維持**し、規約で「advisory lock キー列であって楽観ロックではない」と明記

## 却下した案

- 更新可能な全テーブルに revision: 全書込 SP に expected_revision 必須が付き契約が重い。操作系
  （投票・取下げ・割当）には価値がない
- 今回は置かない: 編集系で lost update が実際に起きる（FAQ Q57 相当）ので既定が要る
- lock_no を lock_key に改名: 型紙と列名が食い違う。名前でなく規約で区別する

## 影響

既存 3 パッケージには遡及しない。新パッケージから適用。型紙の WF006 を XX010 に揃えるかは
workflow-component 側の backlog に持ち越し。
