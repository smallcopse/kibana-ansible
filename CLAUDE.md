# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

Kibana の設定を Kibana REST API 経由でコード管理する Ansible ロール集。
`hosts: localhost` で実行し、対象 Kibana へは SSH ではなく HTTP で接続する
（`ansible.builtin.uri` のみ使用、追加コレクション不要）。

管理対象ごとに 1 ロール:
- `kibana_connectors` — コネクター（Actions/Connectors API）
- `kibana_rules` — アラートルール（Alerting API、Observability + Stack Management）
- `kibana_workflows` — Elastic Workflows（Workflows API、技術プレビュー / Enterprise）
- `kibana_common` — 上記 3 つの共通部品。単体では使わない

## コマンド

前提: このリポジトリには CI もテストスイートも無い。開発機に ansible が入っていない
場合は `pipx install ansible ansible-lint` などで用意する。

```bash
# 構文チェック（依存ロール解決のためリポジトリルートで実行）
ansible-playbook playbooks/site.yml --syntax-check

# Lint
ansible-lint

# ドライラン（実際の Kibana に接続して差分だけ確認。GET は実行される）
ansible-playbook playbooks/manage-rules.yml -e @examples/observability-rules.yml --check -v

# 適用（個別 / 一括）
ansible-playbook playbooks/manage-connectors.yml -e @examples/connectors.yml
ansible-playbook playbooks/site.yml -e @vars/kibana.yml --ask-vault-pass

# 1 リソースだけ試す = examples の vars ファイルで kibana_rules 等のリストを 1 件に絞る
#   （タスク単位のフィルタは無いので、対象リストを短くするのが実質的な最小実行）
```

`ansible.cfg` で `roles_path = roles`（cwd 相対）。必ず**リポジトリルートから**
`ansible-playbook` を実行すること。`playbooks/` に cd すると `kibana_common` が解決できない。

## アーキテクチャ（重要）

### kibana_common による共通化

3 つの管理ロールはすべて「同じ Kibana API を同じ認証で叩く」ため、接続まわりを
`kibana_common` に集約している。各ロールは `meta/main.yml` の `dependencies` で
`kibana_common` を読み込み（→ 接続変数が scope に入る）、処理は `tasks_from` で明示呼び出し:

```yaml
# 各ロール tasks/main.yml の冒頭で 1 回
- ansible.builtin.include_role: { name: kibana_common, tasks_from: connect.yml }

# API を叩くたびに（ループ内で複数回）
- ansible.builtin.include_role: { name: kibana_common, tasks_from: request.yml }
  vars: { kr_name: ..., kr_method: ..., kr_url: ..., kr_status: [...], ... }
```

- `connect.yml` → 認証情報の相互依存 assert + `_kibana_api_base` / `_kibana_headers` を set_fact
  （接続変数の型・choices は `kibana_common/meta/argument_specs.yml` の `connect` で検証）
- `request.yml` → `uri` ラッパ。呼び出し規約は下記（`argument_specs` は無し = 検証されない）
- `include_role`（`public: false`）でも `set_fact` / `register` の結果は host fact として
  呼び出し元に伝播する、という前提で全体が組まれている

### request.yml の呼び出し規約（`kr_*` 変数）

| 変数 | 用途 |
|------|------|
| `kr_name` | タスク表示ラベル（`"POST /api/..."` 形式） |
| `kr_method` / `kr_url` / `kr_status` | HTTP メソッド / URL / 許容ステータスコードのリスト |
| `kr_body` | 送信 dict（省略可） |
| `kr_changed` | `true` で `changed_when: true`。**変更系呼び出しだけ** true にする |
| `kr_check_mode` | 読み取り系（一覧取得・検索）は `false`。`--check` でも実行して差分判定を可能にする |

結果は `kibana_request`（`uri` の戻り値）として register される。

### 各ロール共通のパターン

- 入力はデータ駆動のリスト変数（`kibana_rules` / `kibana_connectors` / `kibana_workflows`）
- ロール固有の `state`（`present` / `absent`）はリスト全体に一括適用
- `tasks/main.yml`（前処理 + ループ）→ `tasks/manage_<x>.yml`（1 件分、`loop` で include_tasks）
- **突合は `name`**。存在すれば PUT、無ければ POST、`absent` なら DELETE
- ループで持ち越す一時 fact（`_rule_actions` など）は `manage_<x>.yml` 冒頭で必ずリセットする
- **変数検証の分担**:
  - ロール変数の型 / choices / 既定値 → `meta/argument_specs.yml`（Ansible が自動検証）
    - 各管理ロールは `main` エントリポイント、`kibana_common` は `connect` エントリポイント
    - リスト変数は `type: list, elements: dict` まで。要素の `options` は定義しない
      （欠けたキーへの `None` 注入で `'key' in item` 判定が壊れるのを避けるため）
  - 要素の必須項目・相互依存（「present なら X 必須」「1つだけ指定」）→ `manage_<x>.yml` の `assert`
  - 認証方式に応じた必須（api_key なら key 必須）→ `kibana_common/tasks/connect.yml` の `assert`
- 差分検出は限定的で、各ロールに「検出できない差分」の注記がある:
  - rules: `actions` のみの変更は非検出 → `kibana_rules_force_update`
  - connectors: `secrets` は Kibana が返さないため非検出 → `kibana_connectors_force_update`
  - workflows: 既存に `yaml` フィールドが無ければ毎回 PUT

### ロール固有の要点

- **kibana_rules**: アクションは `id` か `connector_name` を指定。`connector_name` は
  実行時に `GET /api/actions/connectors` で id 解決（`resolve_connectors.yml`）。
  作成 body は `rule_type_id` / `consumer` / `enabled` を含むが、更新 body は含まない
  （PUT が受け付けない）。有効/無効は `_enable` / `_disable` エンドポイントで既存ルールのみ調整。
- **kibana_connectors**: `is_preconfigured` のコネクターは更新・削除不可 → スキップして警告。
- **kibana_workflows**: API パス（`kibana_workflows_list_path` / `_item_path`）、一覧取得の
  クエリ（`kibana_workflows_list_query`）、バージョンヘッダ（`elastic-api-version`、
  `kibana_extra_headers` 経由で connect.yml に渡す）、一覧のラップキー
  （`kibana_workflows_list_key`）はすべて変数で上書き可能（API がプレビューで変わり得るため）。
  workflow 定義は `definition`(dict) / `yaml`(文字列) / `src`(ファイル) のいずれか 1 つ。
  `_wf_yaml` は `ansible.builtin.unsafe` で包み、body 生成時に workflow 内の `{{ ... }}`
  （Mustache）が再展開されないようにしている。`definition` / `yaml` 直書き時は
  `"{{ '{{ consts.x }}' }}"` のようにエスケープが必要、`src` は不要。

## コーディング規約

- **コメントは日本語**、タスクの `name:` は英語（識別子的な扱い）
- モジュールは完全修飾名（`ansible.builtin.*`）
- 新しい管理対象を追加する = 既存ロールの骨格を踏襲した新ロールを作る
  （`meta/main.yml` で `kibana_common` 依存、`meta/argument_specs.yml` で `main` を定義、
  `tasks/main.yml` で connect → 一覧取得 → ループ、`tasks/manage_<x>.yml` で name 突合 → CRUD、
  `request.yml` を `include_role` で利用）
- `examples/*.yml` は「接続情報 + 定義リスト」を含む extra-vars ファイル。
  シークレットは `{{ vault_* }}` 参照にとどめ、値は書かない（`.gitignore` で `**/vault.yml` 除外）

## 参照

利用者向けの詳細は `README.md` と各ロールの `roles/*/README.md`。
