# 要件定義 — OrgGraph

> **言語:** 日本語 · English: [requirements.md](./requirements.md)

| 項目 | 内容 |
|------|------|
| プロダクト名 | OrgGraph（Temporal Authorization Platform） |
| ドキュメント種別 | 技術要件定義 |
| 関連ドキュメント | [product-requirements-ja.md](./product-requirements-ja.md), [README.md](../../README.md) |
| ステータス | Draft |

---

## 1. 目的

本ドキュメントは OrgGraph の**実装に必要な機能要件・データ要件・API 要件**を定義する。

OrgGraph は認証（Authentication）ではなく**認可（Authorization）**プラットフォームである。Identity Provider（IdP）と連携し、指定時点における「誰が」「どの組織・役割として」「どのリソースに対して」「何をしてよいか」を判定する。

---

## 2. 用語

| 用語 | 定義 |
|------|------|
| Temporal Authorization | 権限・所属・リソースの有効期間を考慮し、**任意の時点**（`effective_at`）で認可を評価する方式 |
| Organization Graph | ユーザー・組織・プロジェクト間の多対多・多様な関係をグラフとして表現したモデル |
| Resource Graph | リソースの所有・分類・有効期間を含むグラフモデル |
| Decision Tree | 認可判定の根拠を User → Organization → Role → Permission → Resource の連鎖として返す構造 |
| effective_at | 認可を評価する基準日時（ISO 8601 UTC） |
| effective_from / effective_to | エンティティまたは関係の有効期間。`effective_to = null` は現在有効を意味する |

---

## 3. スコープ

### 3.1 対象

- ユーザー・組織・ロール・権限・リソースの CRUD
- 時点指定による認可判定（`/authorize`）
- 判定理由の説明（`/explain`）
- 認可判定の監査ログ
- AI Agent 向け REST API / MCP / SDK

### 3.2 対象外

以下は IdP または外部サービスに委譲し、本プロダクトでは提供しない。

- SSO / MFA / パスワード認証 / メール認証
- Identity Provider 機能そのもの

---

## 4. フェーズ別スコープ

README のロードマップに準拠する。各フェーズの詳細要件は後続セクションを参照。

| フェーズ | スコープ | 備考 |
|----------|----------|------|
| **Phase 1** | Google Workspace ログイン、Organization / User / Role / Permission、REST API（CRUD + 基本 authorize） | MVP。Temporal は `effective_from` / `effective_to` フィールドを保持するが、Graph / Explain は最小限 |
| **Phase 2** | Temporal Authorization（時点評価の完全実装） | `effective_at` による履歴評価 |
| **Phase 3** | Organization Graph（全 Relationship 種別） | 兼務・承認者・代理など |
| **Phase 4** | Resource Graph | リソースの時間属性・分類 |
| **Phase 5** | Explainable Authorization | Decision Tree の完全実装 |
| **Phase 6** | Google Workspace Sync | 組織・ユーザー同期 |
| **Phase 7** | SmartHR / Workday / Entra / Okta 連携 | 追加 IdP |
| **Phase 8** | AI Agent Runtime | ランタイム統合 |

---

## 5. ドメインモデル

### 5.1 共通：Temporal 属性

**Phase 2 以降、すべての時系列エンティティおよび関係**に以下を付与する。Phase 1 ではスキーマ上保持し、評価ロジックは Phase 2 で完成させる。

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `effective_from` | `datetime` (ISO 8601 UTC) | ○ | 有効開始日時 |
| `effective_to` | `datetime \| null` | ○ | 有効終了日時。`null` = 無期限（現在有効） |

**不変条件**

- `effective_from <= effective_to`（`effective_to` が非 null の場合）
- 同一ユーザー・同一組織・同一関係種別について、有効期間が重複するレコードは Phase 3 以降で排他制御または明示的な優先度ルールで解決する

---

### 5.2 Identity（Phase 1）

外部 IdP による認証。Phase 1 は Google Workspace のみ。

| 方式 | 要件 |
|------|------|
| OAuth 2.0 | Authorization Code Flow |
| OIDC | `sub`, `email`, `name` を IdP から取得 |
| セッション | API 呼び出しは Bearer Token（JWT）で認証。OrgGraph は JWT の `sub` を内部 `user_id` に解決する |

**受け入れ基準**

- Google Workspace アカウントでログインし、以降の API を呼び出せること
- OrgGraph 自身はパスワードを保持しないこと

---

### 5.3 Organization（Phase 1）

組織単位。階層は Graph Relationship で表現する（Phase 3）。Phase 1 ではフラットな CRUD。

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `id` | `uuid` | ○ | 組織 ID |
| `type` | `enum` | ○ | `company` \| `department` \| `team` \| `project` |
| `name` | `string` | ○ | 表示名 |
| `code` | `string` | - | 社内コード（部署コード等） |
| `parent_id` | `uuid \| null` | - | Phase 1 の簡易親子。Phase 3 以降は Graph に移行 |
| `effective_from` | `datetime` | ○ | |
| `effective_to` | `datetime \| null` | ○ | |

**組織種別**

| type | 用途 |
|------|------|
| `company` | 法人・会社 |
| `department` | 部署 |
| `team` | チーム |
| `project` | プロジェクト（期間限定の横断組織） |

---

### 5.4 User（Phase 1）

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `id` | `uuid` | ○ | 内部ユーザー ID |
| `employee_number` | `string` | - | 社員番号 |
| `email` | `string` | ○ | 一意。IdP と突合 |
| `google_id` | `string` | ○ (Phase 1) | Google `sub` |
| `display_name` | `string` | - | 表示名 |
| `status` | `enum` | ○ | `active` \| `inactive` \| `suspended` |
| `effective_from` | `datetime` | ○ | 在籍開始 |
| `effective_to` | `datetime \| null` | ○ | 退職日等。`null` = 在籍中 |

**不変条件**

- `email` はシステム内で一意
- `status = inactive` のユーザーは `allow = false`（理由: `user_inactive`）

---

### 5.5 Role（Phase 1）

役職・ロール定義（ユーザーへの付与は Assignment で行う）。

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `id` | `uuid` | ○ | |
| `name` | `string` | ○ | 例: `Manager`, `Contract Reader` |
| `description` | `string` | - | |
| `organization_id` | `uuid \| null` | - | 組織スコープのロール。`null` は全社共通 |
| `effective_from` | `datetime` | ○ | |
| `effective_to` | `datetime \| null` | ○ | |

---

### 5.6 Permission（Phase 1）

ロールに紐づく操作許可。

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `id` | `uuid` | ○ | |
| `role_id` | `uuid` | ○ | 所属ロール |
| `action` | `string` | ○ | 例: `read`, `write`, `delete`, `approve` |
| `resource_type` | `string` | ○ | 例: `contract`, `invoice`, `drive_folder` |
| `conditions` | `json` | - | Phase 4 以降。分類・部署等の追加条件 |
| `effective_from` | `datetime` | ○ | |
| `effective_to` | `datetime \| null` | ○ | |

**action 命名規則**

- 小文字スネークケース推奨（`read`, `write`, `delete`, `approve`, `delegate`）
- ワイルドカード `*` は Phase 2 以降で検討

---

### 5.7 Resource（Phase 4、Phase 1 はスキーマのみ）

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `id` | `uuid` | ○ | |
| `type` | `string` | ○ | 例: `drive_folder`, `slack_channel`, `contract` |
| `external_id` | `string` | - | 外部システム上の ID |
| `owner_user_id` | `uuid \| null` | - | オーナー |
| `owner_organization_id` | `uuid \| null` | - | 所有部署 |
| `classification` | `enum` | - | `public` \| `internal` \| `confidential` \| `restricted` |
| `effective_from` | `datetime` | ○ | リソース有効開始 |
| `effective_to` | `datetime \| null` | ○ | リソース有効終了 |

---

### 5.8 Assignment（所属・ロール付与）

ユーザーと組織・ロールの関係。Phase 1 では `MEMBER_OF` + ロール付与を Assignment API で表現。

| 属性 | 型 | 必須 | 説明 |
|------|-----|------|------|
| `id` | `uuid` | ○ | |
| `user_id` | `uuid` | ○ | |
| `organization_id` | `uuid` | ○ | |
| `role_id` | `uuid` | ○ | |
| `relationship` | `enum` | ○ | Phase 3 以降は Graph Relationship と統合 |
| `effective_from` | `datetime` | ○ | |
| `effective_to` | `datetime \| null` | ○ | |

---

## 6. Organization Graph（Phase 3）

組織・ユーザー・プロジェクト間の関係種別。

| Relationship | 主体 | 客体 | 説明 | 例 |
|--------------|------|------|------|-----|
| `BELONGS_TO` | User / Org | Organization | 所属 | ユーザー → 営業部 |
| `REPORTS_TO` | User | User | 報告先 | メンバー → マネージャ |
| `MANAGES` | User | User / Org | 管理権 | マネージャ → チーム |
| `MEMBER_OF` | User | Organization | メンバー（Assignment と同等） | PJ メンバー |
| `OWNER_OF` | User / Org | Resource | リソース所有者 | 契約書オーナー |
| `APPROVER_OF` | User | Resource / Org | 承認者 | 契約承認者 |
| `DELEGATED_TO` | User | User | 権限代理 | 休暇中の代理承認 |
| `PROJECT_MEMBER` | User | Project | PJ メンバー | |
| `PROJECT_OWNER` | User | Project | PJ オーナー | |

**グラフ走査ルール（Phase 3+）**

- 認可評価時、指定 `effective_at` に有効なエッジのみ traversable とする
- 複数パスが存在する場合、いずれか 1 パスで Permission が満たされれば `allow = true`（OR 評価）
- 循環参照は登録時に拒否する

---

## 7. 認可評価

### 7.1 評価フロー

```
1. user_id, resource_id, action, effective_at を受け取る
2. effective_at において user が active か確認
3. effective_at において user の Organization Graph 上の所属・ロールを解決
4. 解決されたロールに紐づく Permission の action / resource_type を照合
5. Resource Graph（Phase 4）でリソースの classification・有効期間を確認
6. 結果 (allow, reason, decision_id) を返却し Audit Log に記録
```

### 7.2 判定結果

| フィールド | 型 | 説明 |
|------------|-----|------|
| `allow` | `boolean` | 許可 / 拒否 |
| `reason` | `string[]` | 人間可読な理由チェーン（Explain 簡易版） |
| `decision_id` | `uuid` | 監査・Explain 用の一意 ID |
| `deny_code` | `string` | 拒否時の機械可読コード（例: `no_matching_permission`, `user_inactive`, `resource_expired`） |

### 7.3 評価例

**入力**

- ユーザー: 2025-02-01 時点で営業部 Manager
- リソース: 2024 年度営業契約（`type=contract`, `classification=confidential`）
- action: `read`
- effective_at: `2025-02-01T00:00:00Z`

**期待出力**

```json
{
  "allow": true,
  "decision_id": "550e8400-e29b-41d4-a716-446655440000",
  "reason": [
    "Sales Department",
    "Manager",
    "Contract Reader",
    "Sales Contract 2024"
  ]
}
```

---

## 8. Explainable Authorization（Phase 5）

### 8.1 Decision Tree 構造

`/explain` は `decision_id` に対応する判定を **木構造**で返す。

```
User (user_id, display_name)
  └─ Organization (org_id, name, effective_from..to)
       └─ Role (role_id, name)
            └─ Permission (permission_id, action, resource_type)
                 └─ Policy (conditions)          ← Phase 4+
                      └─ Resource (resource_id, type, classification)
                           └─ Decision: ALLOW | DENY
```

### 8.2 レスポンススキーマ（概要）

```json
{
  "decision_id": "uuid",
  "allow": true,
  "tree": {
    "node_type": "user",
    "id": "uuid",
    "label": "山田 太郎",
    "children": [
      {
        "node_type": "organization",
        "id": "uuid",
        "label": "営業部",
        "effective_from": "2024-04-01T00:00:00Z",
        "effective_to": "2025-03-31T23:59:59Z",
        "children": [ "..."]
      }
    ]
  }
}
```

**受け入れ基準**

- 任意の `decision_id` に対し、Audit Log に記録された判定と一致する Tree が返ること
- DENY の場合、拒否となった最初のノードに `deny_reason` が付与されること

---

## 9. REST API

ベース URL: `/api/v1`  
認証: `Authorization: Bearer <JWT>`  
Content-Type: `application/json`  
日時: ISO 8601 UTC（例: `2025-02-01T00:00:00Z`）

### 9.1 POST /authorize

指定時点での認可判定。

**Request**

```json
{
  "user_id": "uuid",
  "resource_id": "uuid",
  "action": "read",
  "effective_at": "2025-02-01T00:00:00Z"
}
```

| フィールド | 型 | 必須 | 説明 |
|------------|-----|------|------|
| `user_id` | `uuid` | ○ | 判定対象ユーザー |
| `resource_id` | `uuid` | ○ | 対象リソース |
| `action` | `string` | ○ | 操作 |
| `effective_at` | `datetime` | ○ | 評価基準日時 |

**Response `200`**

```json
{
  "allow": true,
  "decision_id": "uuid",
  "reason": ["Sales Department", "Manager", "Contract Reader"],
  "deny_code": null
}
```

**Error**

| Status | 条件 |
|--------|------|
| `400` | 必須フィールド欠落、`effective_at` 形式不正 |
| `404` | `user_id` または `resource_id` が存在しない |
| `401` | 未認証 |

---

### 9.2 POST /explain

**Request**

```json
{
  "decision_id": "uuid"
}
```

**Response `200`**

```json
{
  "decision_id": "uuid",
  "allow": true,
  "tree": { }
}
```

| Status | 条件 |
|--------|------|
| `404` | `decision_id` が Audit Log に存在しない |

---

### 9.3 CRUD エンドポイント（Phase 1）

| Method | Path | 説明 | Phase |
|--------|------|------|-------|
| `POST` | `/users` | ユーザー作成 | 1 |
| `GET` | `/users/{id}` | ユーザー取得 | 1 |
| `POST` | `/organizations` | 組織作成 | 1 |
| `GET` | `/organizations/{id}` | 組織取得 | 1 |
| `POST` | `/roles` | ロール作成 | 1 |
| `POST` | `/permissions` | 権限作成 | 1 |
| `POST` | `/assignments` | 所属・ロール付与 | 1 |
| `POST` | `/resources` | リソース作成 | 4 |

**共通エラーレスポンス**

```json
{
  "error": {
    "code": "validation_error",
    "message": "effective_from must be before effective_to",
    "details": []
  }
}
```

---

### 9.4 POST /users（例）

**Request**

```json
{
  "email": "taro@example.com",
  "google_id": "1082...",
  "employee_number": "E001",
  "display_name": "山田 太郎",
  "status": "active",
  "effective_from": "2024-04-01T00:00:00Z",
  "effective_to": null
}
```

**Response `201`**

```json
{
  "id": "uuid",
  "email": "taro@example.com",
  "status": "active",
  "created_at": "2025-06-29T00:00:00Z"
}
```

---

### 9.5 POST /assignments（例）

**Request**

```json
{
  "user_id": "uuid",
  "organization_id": "uuid",
  "role_id": "uuid",
  "relationship": "MEMBER_OF",
  "effective_from": "2024-04-01T00:00:00Z",
  "effective_to": "2025-03-31T23:59:59Z"
}
```

---

## 10. AI 向け API（MCP / SDK）

AI Agent が直接呼び出すインターフェース。REST API のラッパーとして実装する。

| 関数 / Tool | 対応 REST | Phase | 説明 |
|-------------|-----------|-------|------|
| `authorize()` | `POST /authorize` | 1 | 認可判定 |
| `explainDecision()` | `POST /explain` | 5 | 判定説明 |
| `searchUser()` | `GET /users` (query) | 1 | ユーザー検索 |
| `searchOrganization()` | `GET /organizations` (query) | 1 | 組織検索 |
| `searchRoleHistory()` | `GET /assignments` (temporal filter) | 2 | ロール履歴 |
| `searchOrganizationHistory()` | Graph query | 3 | 所属履歴 |
| `searchPermission()` | `GET /permissions` (query) | 1 | 権限検索 |
| `searchResource()` | `GET /resources` (query) | 4 | リソース検索 |

### 10.1 MCP Server

- OrgGraph は **MCP Server を標準同梱**する
- Transport: stdio（ローカル）/ SSE（リモート）をサポート
- 上記 Tool を MCP Tool として公開
- 認証: 環境変数または OAuth Token を MCP 接続時に注入

### 10.2 SDK

| 言語 | Phase | 備考 |
|------|-------|------|
| Go | 1 | サーバー実装言語と同一 |
| TypeScript | 2 | AI Agent / Node 向け |
| Python | 2 | AI Agent 向け |
| Java | 3 | エンタープライズ向け |

SDK は以下を提供する。

- 型付き Request / Response
- 自動リトライ（429 / 5xx）
- `effective_at` の Date 型変換

---

## 11. 監査（Audit）

**すべての `/authorize` 呼び出し**を監査ログに永続化する。

| フィールド | 型 | 説明 |
|------------|-----|------|
| `id` | `uuid` | = `decision_id` |
| `timestamp` | `datetime` | 判定実行日時（サーバー時刻） |
| `effective_at` | `datetime` | 評価基準日時 |
| `user_id` | `uuid` | |
| `resource_id` | `uuid` | |
| `action` | `string` | |
| `allow` | `boolean` | 判定結果 |
| `reason` | `string[]` | 理由チェーン |
| `deny_code` | `string \| null` | |
| `agent_id` | `string \| null` | 呼び出し AI Agent 識別子 |
| `tool_name` | `string \| null` | 呼び出し Tool 名（MCP tool name 等） |
| `requester` | `string` | API キー / JWT `sub` / サービスアカウント |

**保持期間**: デフォルト 7 年（設定可能）。改ざん防止のため append-only。

**受け入れ基準**

- 同一 `decision_id` で `/explain` の内容と Audit Log が一致すること
- `agent_id` / `tool_name` は MCP 経由呼び出し時に必須

---

## 12. 非機能要件

| カテゴリ | 要件 |
|----------|------|
| 可用性 | Phase 1: 単一リージョン、99.5% SLA（Self-hosted 時はベストエフォート） |
| レイテンシ | `/authorize` p99 < 200ms（キャッシュなし、Graph 深さ ≤ 5） |
| スケーラビリティ | 10 万ユーザー / 100 万リソース規模を Phase 4 で想定 |
| セキュリティ | TLS 必須、JWT 検証、監査ログへの PII 最小化 |
| 互換性 | REST API は OpenAPI 3.1 で公開 |
| ライセンス | Apache-2.0 |

---

## 13. Phase 1 受け入れ基準（まとめ）

- [ ] Google Workspace OIDC でログインできる
- [ ] User / Organization / Role / Permission / Assignment を CRUD できる
- [ ] `POST /authorize` が `effective_at` を受け取り `allow` / `reason` / `decision_id` を返す
- [ ] すべての authorize 呼び出しが Audit Log に記録される
- [ ] `effective_from` / `effective_to` が全主要エンティティに存在する
- [ ] OpenAPI 仕様書が公開されている

---

## 14. 未決事項

| # | 論点 | 候補 | 決定期限 |
|---|------|------|----------|
| 1 | Permission の `action` 標準語彙 | CASL / custom enum | Phase 1 開始前 |
| 2 | 兼務時のロール優先度 | Union (OR) / Explicit priority | Phase 3 |
| 3 | Resource の外部 ID 同期方式 | Webhook / Polling / MCP | Phase 6 |
| 4 | Graph DB 選定 | PostgreSQL + adjacency / Neo4j | Phase 3 設計時 |
