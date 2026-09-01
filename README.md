# kibana-ansible

Kibana の設定をコードとして管理するための Ansible コンテンツ。

## role 一覧

| role | 目的 |
|------|------|
| [`kibana_rules`](roles/kibana_rules/README.md) | Kibana の **Observability → アラート** と **Stack Management → ルール**（アラートルール）を Kibana API 経由で作成 / 更新 / 削除する。 |

## 構成

```
ansible.cfg
inventory/hosts.ini            # localhost で実行する（Kibana へは HTTP で接続）
playbooks/manage-rules.yml     # kibana_rules role のエントリポイント
examples/                      # extra-vars ファイルのサンプル（接続情報 + ルール）
roles/kibana_rules/            # role 本体
```

## クイックスタート

1. *アラートルール*の管理権限を持つ Kibana API キー（または Basic 認証ユーザー）を用意する。

2. 接続情報とルールを vars ファイルに記述する。
   [`examples/observability-rules.yml`](examples/observability-rules.yml) または
   [`examples/stack-management-rules.yml`](examples/stack-management-rules.yml) を出発点にする。
   シークレットは `ansible-vault` で暗号化したファイルに置く:

   ```bash
   ansible-vault create group_vars/all/vault.yml
   # vault_kibana_api_key: "<エンコード済み API キー>"
   ```

3. 実行:

   ```bash
   ansible-galaxy collection install -r requirements.yml   # 現状は不要
   ansible-playbook playbooks/manage-rules.yml \
     -e @examples/observability-rules.yml \
     --ask-vault-pass --check      # 適用するときは --check を外す
   ```

## 要件

* Ansible 2.14 以上
* Python の `requests` は不要（role は `ansible.builtin.uri` を使用）。

変数の詳細とルール定義スキーマは [role の README](roles/kibana_rules/README.md) を参照。
