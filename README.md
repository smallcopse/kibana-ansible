# kibana-ansible

Kibana の設定をコードとして管理するための Ansible コンテンツ。

## role 一覧

| role | 目的 |
|------|------|
| [`kibana_connectors`](roles/kibana_connectors/README.md) | Kibana のコネクター（Slack / Email / Webhook / PagerDuty / Index / Server log / Jira など）を Kibana API 経由で作成 / 更新 / 削除する。 |
| [`kibana_rules`](roles/kibana_rules/README.md) | Kibana の **Observability → アラート** と **Stack Management → ルール**（アラートルール）を Kibana API 経由で作成 / 更新 / 削除する。 |

両 role は同じ接続変数（`kibana_url` / `kibana_auth_method` など）を使うため、
`group_vars` で設定を共用できます。

## 構成

```
ansible.cfg
inventory/hosts.ini            # localhost で実行する（Kibana へは HTTP で接続）
playbooks/
  manage-connectors.yml        # kibana_connectors role のみ
  manage-rules.yml             # kibana_rules role のみ
  site.yml                     # コネクター -> ルール の順でまとめて適用
examples/                      # extra-vars ファイルのサンプル（接続情報 + 定義）
roles/kibana_connectors/
roles/kibana_rules/
```

## クイックスタート

1. *コネクター*と*アラートルール*の管理権限を持つ Kibana API キー
   （または Basic 認証ユーザー）を用意する。

2. 接続情報と定義を vars ファイルに記述する。examples/ を出発点にする:
   - [`examples/connectors.yml`](examples/connectors.yml)
   - [`examples/observability-rules.yml`](examples/observability-rules.yml)
   - [`examples/stack-management-rules.yml`](examples/stack-management-rules.yml)

   シークレットは `ansible-vault` で暗号化したファイルに置く:

   ```bash
   ansible-vault create group_vars/all/vault.yml
   # vault_kibana_api_key: "<エンコード済み API キー>"
   # vault_slack_webhook_url: "https://hooks.slack.com/services/..."
   ```

3. 実行:

   ```bash
   ansible-galaxy collection install -r requirements.yml   # 現状は不要

   # コネクターとルールをまとめて適用（--check で適用しないプレビュー）
   ansible-playbook playbooks/site.yml -e @vars/kibana.yml --ask-vault-pass --check

   # 個別に実行する場合
   ansible-playbook playbooks/manage-connectors.yml -e @examples/connectors.yml
   ansible-playbook playbooks/manage-rules.yml      -e @examples/observability-rules.yml
   ```

コネクターを先に作成し、ルールのアクションからは `connector_name` で参照します。

## 要件

* Ansible 2.14 以上
* Python の `requests` は不要（role は `ansible.builtin.uri` を使用）。

変数の詳細と定義スキーマは各 role の README を参照:
[kibana_connectors](roles/kibana_connectors/README.md) /
[kibana_rules](roles/kibana_rules/README.md)。
