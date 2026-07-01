# OrgGraph

> AI-native Temporal Authorization Platform

> **Language:** English · [日本語版](#orggraph-1) below

OrgGraph is an open-source **authorization** platform built for the AI Agent era.

Unlike traditional IAM or RBAC, OrgGraph evaluates access using **organization history**, **role history**, and **resource history** — not just who the user is today.

It answers one question:

> **"Was this user allowed to perform this action on this resource at that point in time?"**

**Status:** Planning / requirements phase (no implementation yet)

**Docs:** [Technical requirements](./docs/requirements/requirements.md) · [Product requirements](./docs/requirements/product-requirements.md)

---

## Table of Contents

- [Why OrgGraph?](#why-orggraph)
- [Vision](#vision)
- [Core Concepts](#core-concepts)
- [Features](#features)
- [Roadmap](#roadmap)
- [API](#api)
- [MCP](#mcp)
- [Non-Goals](#non-goals)
- [License](#license)
- [日本語版](#orggraph-1)

---

## Why OrgGraph?

Most authorization systems assume permissions exist **only now**.

```
User → Role → Permission
```

Enterprise reality is messier:

- Employees move between departments
- Managers and approval chains change
- Projects start and end
- AI Agents need to reason about **historical** data access

Example:

```
User

2024-04 ~ 2025-03   Sales Manager
2025-04 ~           AI Strategy Office
```

When an AI Agent asks:

> "Can I summarize the Sales contract from February 2025?"

The decision must use **February 2025**, not today's membership.

That is **Temporal Authorization**.

---

## Vision

Become the standard authorization layer for AI Agents in the enterprise.

```
Google Workspace ─┐
Microsoft Entra  ─┤
Okta             ─┤
                  ├──► OrgGraph ──► AI Agents
SmartHR / Workday─┘              Slack · Drive · GitHub · Salesforce · ERP
```

OrgGraph is **not** an Identity Provider. It is an **Authorization Platform** that sits beside your IdP.

---

## Core Concepts

### Identity

Authentication is delegated to existing providers (Google Workspace, Microsoft Entra ID, Okta, etc.). OrgGraph maps IdP subjects to internal `user_id` and owns all authorization logic.

### Organization Graph

Enterprise structure as a graph — not a flat department tree.

```
Company
 ├── Division
 │     ├── Department
 │     └── Team
 └── Project
```

Supports concurrent posts, managers, approvers, delegation, and multiple assignments (full graph in Phase 3).

### Temporal Authorization

Every entity and relationship carries a validity window:

| Field | Meaning |
|-------|---------|
| `effective_from` | Validity start (ISO 8601 UTC) |
| `effective_to` | Validity end; `null` = currently active |

Evaluation at any historical point uses `effective_at`.

### Resource Graph

Resources are versioned and classified (Phase 4): Drive folders, Slack channels, contracts, invoices, etc. Authorization combines **User Graph × Resource Graph** at `effective_at`.

### Explainable Authorization

Every decision is auditable and explainable as a Decision Tree (Phase 5):

```
ALLOW
  └─ Sales Department
       └─ Manager role
            └─ contract:read permission
                 └─ Target contract (Feb 2025)
```

---

## Features

| Area | Description |
|------|-------------|
| Temporal RBAC | Point-in-time permission evaluation via `effective_at` |
| Organization Graph | Rich org relationships beyond a simple tree |
| Resource Graph | Temporal attributes and classification on resources |
| Explainable Authorization | `/explain` returns a Decision Tree |
| REST API | `/api/v1` — CRUD + authorize + explain |
| MCP Server | stdio (local) and SSE (remote) transports |
| SDKs | Go (Phase 1), TypeScript + Python (Phase 2) |
| Audit Logs | Append-only; 7-year retention by default |

---

## Roadmap

| Phase | Scope |
|-------|-------|
| **Phase 1** | Google Workspace OIDC; User / Organization / Role / Permission / Assignment CRUD; `POST /authorize`; Audit Log; Go SDK; OpenAPI 3.1 |
| **Phase 2** | Full temporal evaluation via `effective_at`; TypeScript + Python SDKs |
| **Phase 3** | Organization Graph — concurrent posts, delegation, approvers, graph traversal |
| **Phase 4** | Resource Graph — temporal attributes and classification |
| **Phase 5** | Explainable Authorization — full Decision Tree via `/explain` |
| **Phase 6** | Google Workspace sync |
| **Phase 7** | SmartHR / Workday / Entra / Okta integration |
| **Phase 8** | AI Agent Runtime integration |

---

## API

Base URL: `/api/v1` · Auth: `Authorization: Bearer <JWT>`

### Authorize

```http
POST /api/v1/authorize
```

```json
{
  "user_id": "usr_...",
  "resource_id": "res_...",
  "action": "read",
  "effective_at": "2025-02-01T00:00:00Z"
}
```

```json
{
  "allow": true,
  "reason": [
    "Sales Department",
    "Manager",
    "contract:read"
  ],
  "decision_id": "dec_...",
  "deny_code": null
}
```

See [requirements.md](./docs/requirements/requirements.md) for full schemas, error shapes, and CRUD endpoints.

---

## MCP

OrgGraph ships an MCP Server (wrappers over the REST API).

Planned tools:

| Tool | Purpose |
|------|---------|
| `authorize()` | Evaluate access at `effective_at` |
| `explainDecision()` | Return Decision Tree for a prior decision |
| `searchOrganization()` | Query org graph |
| `searchUser()` | Query users and membership history |
| `searchRoleHistory()` | Query role assignments over time |
| `searchPermission()` | Query permissions bound to roles |
| `searchResource()` | Query resources and attributes |

Transports: **stdio** (local) · **SSE** (remote)

---

## Non-Goals

OrgGraph does **not** provide:

- Password authentication · MFA · SSO
- Identity Provider functionality
- Email authentication

Use Google Workspace, Entra ID, Okta, or similar for identity.

---

## License

[Apache-2.0](./LICENSE)

---

# OrgGraph

> AI ネイティブな時間軸認可プラットフォーム

> **言語:** 日本語 · English: [above](#orggraph)

OrgGraph は、AI Agent 時代のためのオープンソース **認可（Authorization）** プラットフォームです。

従来の IAM や RBAC とは異なり、**組織の履歴**・**ロールの履歴**・**リソースの履歴**を使ってアクセスを評価します — 「今のユーザー」だけでは判断しません。

答える問いは一つです:

> **「その時点で、このユーザーはこのリソースに対してその操作をしてよかったか？」**

**ステータス:** 企画 / 要件定義フェーズ（実装は未着手）

**ドキュメント:** [技術要件](./docs/requirements/requirements-ja.md) · [プロダクト要件](./docs/requirements/product-requirements-ja.md)

---

## 目次

- [なぜ OrgGraph か](#なぜ-orggraph-か)
- [ビジョン](#ビジョン)
- [コア概念](#コア概念)
- [機能](#機能)
- [ロードマップ](#ロードマップ)
- [API](#api-1)
- [MCP](#mcp-1)
- [非目標](#非目標)
- [ライセンス](#ライセンス)

---

## なぜ OrgGraph か

多くの認可システムは、権限が **今だけ** 存在すると仮定しています。

```
User → Role → Permission
```

しかし企業の現実はもっと複雑です:

- 社員は部署を異動する
- マネージャーや承認フローは変わる
- プロジェクトは始まり終わる
- AI Agent は **過去の** データアクセスを判断する必要がある

例:

```
User

2024-04 ~ 2025-03   営業部マネージャー
2025-04 ~           AI 推進室
```

AI Agent がこう聞いたとき:

> 「2025年2月の営業契約書を要約してよいか？」

判断に使うのは **2025年2月時点** の所属であり、今日の所属ではありません。

これが **Temporal Authorization（時間軸認可）** です。

---

## ビジョン

エンタープライズにおける AI Agent の標準認可レイヤーになる。

```
Google Workspace ─┐
Microsoft Entra  ─┤
Okta             ─┤
                  ├──► OrgGraph ──► AI Agents
SmartHR / Workday─┘              Slack · Drive · GitHub · Salesforce · ERP
```

OrgGraph は **Identity Provider ではありません**。IdP の横に置く **認可プラットフォーム** です。

---

## コア概念

### Identity（アイデンティティ）

認証は既存プロバイダ（Google Workspace、Microsoft Entra ID、Okta 等）に委譲します。OrgGraph は IdP の subject を内部 `user_id` にマッピングし、認可ロジックをすべて保持します。

### Organization Graph（組織グラフ）

フラットな部署ツリーではなく、グラフとして表現する企業構造。

```
Company
 ├── Division
 │     ├── Department
 │     └── Team
 └── Project
```

兼務、マネージャー、承認者、委任、複数所属などをサポート（完全なグラフは Phase 3）。

### Temporal Authorization（時間軸認可）

すべてのエンティティと関係に有効期間を持たせます:

| Field | 意味 |
|-------|------|
| `effective_from` | 有効開始（ISO 8601 UTC） |
| `effective_to` | 有効終了; `null` = 現在有効 |

任意の過去時点での評価には `effective_at` を使います。

### Resource Graph（リソースグラフ）

リソースも版管理・分類されます（Phase 4）: Drive フォルダ、Slack チャンネル、契約書、請求書など。認可は `effective_at` における **User Graph × Resource Graph** の組み合わせで評価します。

### Explainable Authorization（説明可能な認可）

すべての判断は Decision Tree として監査・説明可能（Phase 5）:

```
ALLOW
  └─ 営業部
       └─ Manager ロール
            └─ contract:read 権限
                 └─ 対象契約書（2025年2月）
```

---

## 機能

| 領域 | 説明 |
|------|------|
| Temporal RBAC | `effective_at` による時点評価 |
| Organization Graph | 単純なツリーを超える組織関係 |
| Resource Graph | リソースの時間属性と分類 |
| Explainable Authorization | `/explain` で Decision Tree を返却 |
| REST API | `/api/v1` — CRUD + authorize + explain |
| MCP Server | stdio（ローカル）と SSE（リモート） |
| SDK | Go（Phase 1）、TypeScript + Python（Phase 2） |
| Audit Logs | 追記専用; デフォルト 7 年保持 |

---

## ロードマップ

| Phase | スコープ |
|-------|----------|
| **Phase 1** | Google Workspace OIDC; User / Organization / Role / Permission / Assignment CRUD; `POST /authorize`; Audit Log; Go SDK; OpenAPI 3.1 |
| **Phase 2** | `effective_at` による完全な時間軸評価; TypeScript + Python SDK |
| **Phase 3** | Organization Graph — 兼務、委任、承認者、グラフ走査 |
| **Phase 4** | Resource Graph — 時間属性と分類 |
| **Phase 5** | Explainable Authorization — `/explain` による完全な Decision Tree |
| **Phase 6** | Google Workspace 同期 |
| **Phase 7** | SmartHR / Workday / Entra / Okta 連携 |
| **Phase 8** | AI Agent Runtime 連携 |

---

## API

Base URL: `/api/v1` · 認証: `Authorization: Bearer <JWT>`

### Authorize

```http
POST /api/v1/authorize
```

```json
{
  "user_id": "usr_...",
  "resource_id": "res_...",
  "action": "read",
  "effective_at": "2025-02-01T00:00:00Z"
}
```

```json
{
  "allow": true,
  "reason": [
    "Sales Department",
    "Manager",
    "contract:read"
  ],
  "decision_id": "dec_...",
  "deny_code": null
}
```

詳細スキーマ・エラー形式・CRUD エンドポイントは [requirements-ja.md](./docs/requirements/requirements-ja.md) を参照。

---

## MCP

OrgGraph は MCP Server を同梱（REST API のラッパー）。

予定ツール:

| Tool | 用途 |
|------|------|
| `authorize()` | `effective_at` でアクセス評価 |
| `explainDecision()` | 過去の判断の Decision Tree を返却 |
| `searchOrganization()` | 組織グラフ検索 |
| `searchUser()` | ユーザーと所属履歴の検索 |
| `searchRoleHistory()` | ロール割当の時系列検索 |
| `searchPermission()` | ロールに紐づく権限の検索 |
| `searchResource()` | リソースと属性の検索 |

トランスポート: **stdio**（ローカル）· **SSE**（リモート）

---

## 非目標

OrgGraph が **提供しない** もの:

- パスワード認証 · MFA · SSO
- Identity Provider 機能
- メール認証

アイデンティティは Google Workspace、Entra ID、Okta 等を利用してください。

---

## ライセンス

[Apache-2.0](./LICENSE)
