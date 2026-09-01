# kibana_workflows

**Elastic Workflows（Kibana Workflows）**の定義を Kibana Workflows API 経由で
作成・管理する Ansible role です。

`kibana_rules` / `kibana_connectors` role と同じ接続変数を使うので、`group_vars` で
設定を共用できます。

> **注意**: Workflows は技術プレビュー段階の機能で、**Enterprise ライセンス**が必要です。
> API のパス・バージョンや workflow スキーマは今後変わる可能性があります。
> この role は API のパス（`kibana_workflows_list_path` / `kibana_workflows_item_path`）と
> バージョンヘッダ（`kibana_workflows_api_version`）を変数で上書きできるようにしています。
> 事前に Kibana 側で Workflows を有効化してください（`kibana.yml` の
> `workflows:experimentalFeatures` など）。

## 動作

`kibana_workflows` の各エントリ（`name` で突合）に対して:

| 状況 | 実行内容 |
|------|----------|
| `state: present` かつ workflow が存在しない | `POST {{ kibana_workflows_item_path }}` |
| `state: present` かつ存在、定義に差分 | `PUT {{ kibana_workflows_item_path }}/{id}` |
| `state: present` かつ `kibana_workflows_force_update: true` | 差分が無くても `PUT` |
| `state: absent` かつ workflow が存在する | `DELETE {{ kibana_workflows_item_path }}/{id}` |

リクエストボディは `{ "yaml": "<workflow YAML 全体>" }` です。

## role 変数

### 接続

接続変数は依存ロール [`kibana_common`](../kibana_common/README.md) で定義しています
（`kibana_url` / `kibana_space` / `kibana_validate_certs` / `kibana_request_timeout` /
`kibana_auth_method` / `kibana_api_key` / `kibana_username` / `kibana_password`）。

### Workflows API / 動作

| 変数 | 既定値 | 備考 |
|------|--------|------|
| `kibana_workflows_api_version` | `2023-10-31` | `elastic-api-version` ヘッダの値。 |
| `kibana_workflows_list_path` | `/api/workflows` | 一覧取得 (GET) のパス。 |
| `kibana_workflows_item_path` | `/api/workflows/workflow` | 作成 (POST) / 更新・削除 (`…/{id}`) のパス。 |
| `kibana_workflows_list_query` | `?limit=1000` | 一覧取得 URL のクエリ文字列。全件取れるページサイズを指定。API が別のパラメータ名なら上書き（例 `?perPage=1000`）。 |
| `kibana_workflows_list_key` | `results` | 一覧レスポンスで配列が入っているキー。配列そのものなら無視。 |
| `kibana_workflows_state` | `present` | `present` または `absent`。 |
| `kibana_workflows_force_update` | `false` | 差分が無くても常に `PUT`。 |
| `kibana_workflows` | `[]` | workflow 定義のリスト（下記）。 |

### workflow 定義のスキーマ

各エントリは `name` に加えて、`definition` / `yaml` / `src` の**いずれか 1 つ**を指定します。

```yaml
kibana_workflows:
  # (A) definition: dict で書く。name は自動でエントリの name に揃えられる
  - name: "my-workflow"
    definition:
      version: "1"
      enabled: true
      triggers:
        - type: manual
      steps:
        - name: step1
          type: console
          with: { message: "hi" }

  # (B) yaml: workflow YAML を文字列でそのまま渡す
  - name: "hello-world"
    yaml: |
      version: "1"
      name: hello-world
      enabled: false
      triggers: [{ type: manual }]
      steps: [{ name: s, type: console, with: { message: hi } }]

  # (C) src: 別ファイルの YAML を読み込む（コントローラー上のパス）
  - name: "incident-enrichment"
    src: "files/workflows/incident-enrichment.yml"
```

workflow YAML の主な top-level フィールド: `version` / `name` / `description` /
`enabled` / `consts` / `inputs` / `triggers` / `settings` / `steps`。

**Mustache 変数のエスケープ**: workflow YAML では `{{ consts.x }}` や
`{{ steps.a.output }}` のような Mustache 記法を多用します。これを `definition` や
`yaml` に直接書くと Ansible が変数として展開しようとしてエラーになります。
`definition` / `yaml` を使う場合は `"{{ '{{ consts.x }}' }}"` のように
エスケープしてください。**`src`（外部ファイル）はそのまま読み込むためエスケープ不要**で、
複雑な workflow は `src` を使うのがおすすめです。

## 差分検出について

既存 workflow のレスポンスに `yaml` フィールドがある場合、双方を `from_yaml` で
パースして dict として比較します（書式の違いによる誤検知を避けるため）。
比較できない場合は安全側に倒して毎回 `PUT`（冪等）を実行します。

## 実行例

```bash
ansible-playbook playbooks/manage-workflows.yml -e @examples/workflows.yml
# プレビューのみ
ansible-playbook playbooks/manage-workflows.yml -e @examples/workflows.yml --check
```

## ライセンス

MIT
