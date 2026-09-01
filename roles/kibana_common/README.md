# kibana_common

`kibana_connectors` / `kibana_rules` / `kibana_workflows` が共通で使う部品ロール。
**単体では使いません**（各ロールの `meta/main.yml` の `dependencies` に入っています）。

## 提供するもの

### 接続変数（`defaults/main.yml`）

Kibana 接続系の変数の唯一の定義場所。各ロールはこれらを再定義しません。

| 変数 | 既定値 | 備考 |
|------|--------|------|
| `kibana_url` | `http://localhost:5601` | ベース URL。末尾 `/api` は不要。 |
| `kibana_space` | `""` | スペース id。空はデフォルト。 |
| `kibana_validate_certs` | `true` | 自己署名証明書なら `false`。 |
| `kibana_request_timeout` | `30` | 秒。 |
| `kibana_auth_method` | `api_key` | `api_key` または `basic`。 |
| `kibana_api_key` | `""` | エンコード済み API キー（base64 `id:api_key`）。 |
| `kibana_username` / `kibana_password` | `""` | `basic` のとき使用。 |
| `kibana_extra_headers` | `{}` | 全リクエストに足す追加ヘッダ。呼び出し側が上書き。 |

### `tasks/connect.yml`

接続チェックと、以下の fact 生成。各ロールの `tasks/main.yml` 冒頭で 1 回呼びます。

```yaml
- name: Set up Kibana connection
  ansible.builtin.include_role:
    name: kibana_common
    tasks_from: connect.yml
  # 追加ヘッダが必要なら:
  # vars:
  #   kibana_extra_headers: { elastic-api-version: "2023-10-31" }
```

生成される fact:

| fact | 内容 |
|------|------|
| `_kibana_api_base` | スペースを考慮した API のベース URL |
| `_kibana_headers` | 全リクエスト共通の HTTP ヘッダ（`kbn-xsrf` / `Content-Type` / 認証 / 追加ヘッダ） |

### `tasks/request.yml`

`ansible.builtin.uri` のラッパ。認証・TLS・タイムアウト・ヘッダの指定を 1 か所に集約。

```yaml
- name: "POST something"
  ansible.builtin.include_role:
    name: kibana_common
    tasks_from: request.yml
  vars:
    kr_name: "POST /api/..."      # タスクの表示ラベル
    kr_method: POST               # GET | POST | PUT | DELETE
    kr_url: "{{ _kibana_api_base }}/api/..."
    kr_body: { ... }              # 任意
    kr_status: [200]              # 許容ステータスコード
    kr_changed: true              # 任意（既定 false）
    kr_check_mode: false          # 任意（読み取り系で --check 時も実行）
```

結果は `kibana_request`（`uri` の戻り値）として register され、include のあとで参照できます。

## 設計メモ

- 3 ロールは同じ Kibana API を同じ認証モデルで叩くファミリーのため、接続変数と
  接続処理・リクエスト処理をこのロールに集約している（重複とドリフトの排除）。
- API 呼び出しごとに `include_role ... tasks_from` するオーバーヘッドは、
  1 回の実行で数十〜百数十回程度なら実用上問題にならない。
- さらに規模が大きくなり、check-mode / diff / 冪等性をきれいに扱いたくなったら、
  カスタムモジュール（`module_utils/kibana.py`）を持つ collection 化を検討する。

## ライセンス

MIT
