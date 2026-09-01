# kibana_connectors

**Kibana のコネクター**（Slack、Email、Webhook、PagerDuty、Index、Server log、
Jira など）を Kibana の Actions/Connectors API 経由で作成・管理する Ansible role です。

`kibana_rules` role と同じ接続変数（`kibana_url` / `kibana_auth_method` など）を
使うので、`group_vars` で設定を共用できます。コネクターを先に作成し、
そのあと `kibana_rules` でルールのアクションから `connector_name` で参照する、
という使い方を想定しています。

## 動作

`kibana_connectors` の各エントリ（対象スペース内で `name` により突合）に対して:

| 状況 | 実行内容 |
|------|----------|
| `state: present` かつコネクターが存在しない | `POST /api/actions/connector` |
| `state: present` かつ存在、name / config に差分（または `is_missing_secrets`） | `PUT /api/actions/connector/{id}` |
| `state: present` かつ `kibana_connectors_force_update: true` | 差分が無くても `PUT` |
| `state: absent` かつコネクターが存在する | `DELETE /api/actions/connector/{id}` |

preconfigured コネクター（`kibana.yml` で定義したもの）は API から更新・削除
できないため、スキップして警告を出します。

冪等です。変更のない状態で再実行すると `ok` になります。

## 要件

* Ansible 2.14 以上（`ansible.builtin` モジュールのみ）。
* 同梱の [`kibana_common`](../kibana_common/README.md) ロール（`meta` の依存）。
* **Management → コネクター**の権限を持つ Kibana ユーザー / API キー。

## role 変数

型・選択肢・既定値は [`meta/argument_specs.yml`](meta/argument_specs.yml) で定義しており、
実行前に Ansible が自動検証します。

### 接続

接続変数は依存ロール [`kibana_common`](../kibana_common/README.md) で定義しています
（`kibana_url` / `kibana_space` / `kibana_validate_certs` / `kibana_request_timeout` /
`kibana_auth_method` / `kibana_api_key` / `kibana_username` / `kibana_password`）。

### 動作

| 変数 | 既定値 | 備考 |
|------|--------|------|
| `kibana_connectors_state` | `present` | `present` または `absent`（リスト全体に適用）。 |
| `kibana_connectors_force_update` | `false` | 差分が無くても常に `PUT`。secrets の入れ替えに使う。 |
| `kibana_connectors` | `[]` | コネクター定義のリスト（下記）。 |

### コネクター定義のスキーマ

```yaml
kibana_connectors:
  - name: "Ops Slack"                 # 必須、突合キー（一意にする）
    connector_type_id: ".slack_api"   # 必須
    config: {}                        # 型ごとの設定（任意。型により不要）
    secrets:                          # トークン等（任意。型により不要）
      token: "{{ vault_slack_token }}"
```

`config` / `secrets` は加工せずそのまま Kibana に送られます。対象の型に対する
正確なキーは
[Kibana のコネクター一覧](https://www.elastic.co/guide/en/kibana/current/action-types.html)
を参照してください。

主な `connector_type_id`:

| 種別 | `connector_type_id` | 主な `config` | 主な `secrets` |
|------|---------------------|---------------|----------------|
| Slack (Webhook) | `.slack` | – | `webhookUrl` |
| Slack API | `.slack_api` | `allowedChannels` | `token` |
| Email | `.email` | `from`, `host`, `port`, `secure`, `hasAuth` | `user`, `password` |
| Webhook (Basic) | `.webhook` | `url`, `method`, `headers`, `hasAuth`, `authType: webhook-authentication-basic` | `user`, `password` |
| Webhook (OAuth2) | `.webhook` | `authType: webhook-authentication-oauth2`, `accessTokenUrl`, `clientId`, `scope`, `additionalFields` | `clientSecret` |
| PagerDuty | `.pagerduty` | `apiUrl` | `routingKey` |
| Index | `.index` | `index`, `refresh` | – |
| Server log | `.server-log` | – | – |
| Jira | `.jira` | `apiUrl`, `projectKey` | `email`, `apiToken` |
| Opsgenie | `.opsgenie` | `apiUrl` | `apiKey` |
| Microsoft Teams | `.teams` | – | `webhookUrl` |
| ServiceNow ITSM | `.servicenow` | `apiUrl`, `usesTableApi` | `username`, `password` |

## secrets の扱いに関する注意

Kibana は API レスポンスで `secrets` を返しません。そのため secrets の差分は
検出できず、`config` と `name` のみで差分判定します。

* secrets（トークン、パスワード等）を更新したいときは、その実行で
  `kibana_connectors_force_update: true` を指定してください。
* コネクターが `is_missing_secrets: true` の状態のときは、差分ありとみなして
  自動的に `PUT`（secrets 再設定）を試みます。
* `config` は Kibana 側で正規化された値が返るため、まれに差分ありと判定されて
  `PUT` が走ることがあります（`PUT` は冪等なので実害はありません）。

## 実行例

```bash
ansible-playbook playbooks/manage-connectors.yml -e @examples/connectors.yml
# プレビューのみ
ansible-playbook playbooks/manage-connectors.yml -e @examples/connectors.yml --check
```

## ライセンス

MIT
