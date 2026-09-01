# kibana_rules

**Kibana のアラートルール**を作成・管理する Ansible role です。
**Observability → アラート**のルール（カスタムしきい値、ログしきい値、SLO バーンレート、
APM、Uptime など）と、**Stack Management → ルール**（Elasticsearch クエリ、
インデックスしきい値、Transform ヘルスなど）の両方を、Kibana Alerting API 経由で扱います。

この role はデータ駆動です。作成したいルールを `kibana_rules` に記述すると、
role が Kibana をその状態に収束させます。

## 動作

`kibana_rules` の各エントリ（対象スペース内で `name` により突合）に対して:

| 状況 | 実行内容 |
|------|----------|
| `state: present` かつルールが存在しない | `POST /api/alerting/rule` |
| `state: present` かつ存在、name/tags/schedule/params に差分 | `PUT /api/alerting/rule/{id}` |
| `state: present` かつ `enabled` が異なる | `POST .../_enable` または `.../_disable` |
| `state: absent` かつルールが存在する | `DELETE /api/alerting/rule/{id}` |

冪等です。変更のない状態で再実行すると `ok` になります。

## 要件

* Ansible 2.14 以上（使用するのは `ansible.builtin` モジュールのみ。追加コレクション不要）。
* Ansible コントローラーから Kibana へネットワーク到達できること
  （role は既定で `hosts: localhost` で実行され、Kibana へは SSH ではなく HTTP で接続します）。
* **Management → アラートルール**権限と、管理するルールタイプに対応する機能権限
  （APM、Infrastructure、Logs、SLOs、Stack Rules など）を持つ Kibana ユーザー / API キー。

## role 変数

### 接続

| 変数 | 既定値 | 備考 |
|------|--------|------|
| `kibana_url` | `http://localhost:5601` | ベース URL。末尾に `/api` は付けない。 |
| `kibana_space` | `""` | スペース id。空はデフォルトスペース。 |
| `kibana_validate_certs` | `true` | 自己署名証明書の場合は `false`。 |
| `kibana_request_timeout` | `30` | 秒。 |
| `kibana_auth_method` | `api_key` | `api_key` または `basic`。 |
| `kibana_api_key` | `""` | **エンコード済み**の API キー（base64 `id:api_key`）。 |
| `kibana_username` / `kibana_password` | `""` | `kibana_auth_method: basic` のときに使用。 |

### 動作

| 変数 | 既定値 | 備考 |
|------|--------|------|
| `kibana_rules_state` | `present` | `present` または `absent`（リスト全体に適用）。 |
| `kibana_rules_resolve_connectors` | `true` | アクション内の `connector_name` を `GET /api/actions/connectors` で id に解決する。 |
| `kibana_rules_force_update` | `false` | 差分が無くても常に `PUT` する。 |
| `kibana_rules` | `[]` | ルール定義のリスト（下記）。 |

### ルール定義のスキーマ

```yaml
kibana_rules:
  - name: "[o11y] High CPU - prod"        # 必須、突合キー（一意にする）
    rule_type_id: "observability.rules.custom_threshold"  # 必須
    consumer: "observability"             # 必須: observability | infrastructure |
                                          #   logs | apm | slo | stackAlerts | alerts
    enabled: true                         # 既定 true
    schedule:
      interval: "1m"                      # 既定 {interval: "1m"}
    tags: ["prod", "infra"]               # 既定 []
    params: {}                            # ルールタイプ固有。そのまま Kibana に渡す
    actions:                              # 既定 []
      - group: "custom_threshold.fired"
        connector_name: "Ops Slack"       # id に解決される。`id:` を直接指定してもよい
        params:
          message: "{{ '{{context.reason}}' }}"
        frequency:
          notify_when: "onActionGroupChange"
          throttle: null
          summary: false
    # 任意の旧形式のルールレベル通知（アクションごとの `frequency` を推奨）:
    # notify_when: "onActionGroupChange"
    # throttle: "1h"
```

`params` は加工せずそのまま Kibana に送られます。対象の `rule_type_id` に対する
正確な形は、既存ルールの API レスポンス（`GET /api/alerting/rules/_find`）や
Kibana UI の「Inspect」表示からコピーしてください。

主な `rule_type_id`:

| 種別 | `rule_type_id` | `consumer` |
|------|----------------|------------|
| カスタムしきい値 | `observability.rules.custom_threshold` | `observability` |
| メトリックしきい値 | `metrics.alert.threshold` | `infrastructure` |
| インベントリ | `metrics.alert.inventory.threshold` | `infrastructure` |
| ログしきい値 | `logs.alert.document.count` | `logs` |
| SLO バーンレート | `slo.rules.burnRate` | `slo` |
| APM レイテンシしきい値 | `apm.transaction_duration` | `apm` |
| APM エラー数 | `apm.error_rate` | `apm` |
| Elasticsearch クエリ | `.es-query` | `stackAlerts` |
| インデックスしきい値 | `.index-threshold` | `stackAlerts` |
| Transform ヘルス | `transform_health` | `stackAlerts` |

## 差分検出の注意

差分は `name` / `tags` / `schedule.interval` / `params` で比較します。
**`actions` だけ**が変わった場合は検出されません。別のフィールドを変更するか、
その実行で `kibana_rules_force_update: true` を指定してください。
なお `params` は Kibana 側で正規化された値が返るため、まれに差分ありと判定されて
`PUT` が走ることがあります（`PUT` は冪等なので実害はありません）。

## 実行例

```bash
ansible-playbook playbooks/manage-rules.yml -e @examples/observability-rules.yml
# プレビューのみ
ansible-playbook playbooks/manage-rules.yml -e @examples/observability-rules.yml --check
```

Observability と Stack Management の完全なルールセットは
[`examples/`](../../examples/) を参照してください。

## ライセンス

MIT
