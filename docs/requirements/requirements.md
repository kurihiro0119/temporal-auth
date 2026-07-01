# Requirements — OrgGraph

> **Language:** English · Japanese: [requirements-ja.md](./requirements-ja.md)

| Field | Value |
|-------|-------|
| Product | OrgGraph (Temporal Authorization Platform) |
| Document type | Technical requirements |
| Related docs | [product-requirements.md](./product-requirements.md), [README.md](../../README.md) |
| Status | Draft |

---

## 1. Purpose

This document defines the **functional, data, and API requirements** needed to implement OrgGraph.

OrgGraph is an **authorization** platform, not an authentication platform. It integrates with Identity Providers (IdPs) and evaluates, at a specified point in time, **who** may perform **what action** on **which resource** under **which organization and role**.

---

## 2. Terminology

| Term | Definition |
|------|------------|
| Temporal Authorization | Authorization evaluated at **any point in time** (`effective_at`), considering validity periods of permissions, memberships, and resources |
| Organization Graph | A graph model representing diverse many-to-many relationships among users, organizations, and projects |
| Resource Graph | A graph model including resource ownership, classification, and validity periods |
| Decision Tree | A structure returning the authorization rationale as User → Organization → Role → Permission → Resource |
| effective_at | The reference datetime for authorization evaluation (ISO 8601 UTC) |
| effective_from / effective_to | Validity period of an entity or relationship. `effective_to = null` means currently active |

---

## 3. Scope

### 3.1 In Scope

- CRUD for users, organizations, roles, permissions, and resources
- Point-in-time authorization (`/authorize`)
- Decision explanation (`/explain`)
- Audit logs for authorization decisions
- REST API / MCP / SDK for AI Agents

### 3.2 Out of Scope

The following are delegated to IdPs or external services and are **not** provided by this product:

- SSO / MFA / password authentication / email authentication
- Identity Provider functionality itself

---

## 4. Phase Scope

Aligned with the README roadmap. Detailed requirements for each phase are in the sections below.

| Phase | Scope | Notes |
|-------|-------|-------|
| **Phase 1** | Google Workspace login, Organization / User / Role / Permission, REST API (CRUD + basic authorize) | MVP. Stores `effective_from` / `effective_to`; Graph / Explain are minimal |
| **Phase 2** | Temporal Authorization (full point-in-time evaluation) | Historical evaluation via `effective_at` |
| **Phase 3** | Organization Graph (all relationship types) | Concurrent posts, approvers, delegation, etc. |
| **Phase 4** | Resource Graph | Temporal attributes and classification for resources |
| **Phase 5** | Explainable Authorization | Full Decision Tree implementation |
| **Phase 6** | Google Workspace Sync | Organization and user sync |
| **Phase 7** | SmartHR / Workday / Entra / Okta integration | Additional IdPs |
| **Phase 8** | AI Agent Runtime | Runtime integration |

---

## 5. Domain Model

### 5.1 Common: Temporal Attributes

**From Phase 2 onward, all temporal entities and relationships** include the following. In Phase 1, the schema holds these fields; evaluation logic is completed in Phase 2.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `effective_from` | `datetime` (ISO 8601 UTC) | Yes | Validity start |
| `effective_to` | `datetime \| null` | Yes | Validity end. `null` = open-ended (currently active) |

**Invariants**

- `effective_from <= effective_to` when `effective_to` is non-null
- For the same user, organization, and relationship type, overlapping validity periods are resolved via exclusive control or explicit priority rules from Phase 3 onward

---

### 5.2 Identity (Phase 1)

Authentication via external IdP. Phase 1 supports Google Workspace only.

| Method | Requirement |
|--------|-------------|
| OAuth 2.0 | Authorization Code Flow |
| OIDC | Obtain `sub`, `email`, `name` from IdP |
| Session | API calls authenticated via Bearer Token (JWT). OrgGraph resolves JWT `sub` to internal `user_id` |

**Acceptance criteria**

- Users can log in with a Google Workspace account and call subsequent APIs
- OrgGraph does not store passwords

---

### 5.3 Organization (Phase 1)

Organizational unit. Hierarchy is expressed via Graph relationships (Phase 3). Phase 1 uses flat CRUD.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `uuid` | Yes | Organization ID |
| `type` | `enum` | Yes | `company` \| `department` \| `team` \| `project` |
| `name` | `string` | Yes | Display name |
| `code` | `string` | No | Internal code (e.g. department code) |
| `parent_id` | `uuid \| null` | No | Simple parent in Phase 1. Migrated to Graph from Phase 3 |
| `effective_from` | `datetime` | Yes | |
| `effective_to` | `datetime \| null` | Yes | |

**Organization types**

| type | Purpose |
|------|---------|
| `company` | Legal entity / company |
| `department` | Department |
| `team` | Team |
| `project` | Time-bound cross-functional organization |

---

### 5.4 User (Phase 1)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `uuid` | Yes | Internal user ID |
| `employee_number` | `string` | No | Employee number |
| `email` | `string` | Yes | Unique. Matched against IdP |
| `google_id` | `string` | Yes (Phase 1) | Google `sub` |
| `display_name` | `string` | No | Display name |
| `status` | `enum` | Yes | `active` \| `inactive` \| `suspended` |
| `effective_from` | `datetime` | Yes | Employment start |
| `effective_to` | `datetime \| null` | Yes | Termination date, etc. `null` = currently employed |

**Invariants**

- `email` is unique within the system
- Users with `status = inactive` always receive `allow = false` (reason: `user_inactive`)

---

### 5.5 Role (Phase 1)

Role definition. Assignment to users is done via the Assignment API.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `uuid` | Yes | |
| `name` | `string` | Yes | e.g. `Manager`, `Contract Reader` |
| `description` | `string` | No | |
| `organization_id` | `uuid \| null` | No | Organization-scoped role. `null` = company-wide |
| `effective_from` | `datetime` | Yes | |
| `effective_to` | `datetime \| null` | Yes | |

---

### 5.6 Permission (Phase 1)

Operation permission bound to a role.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `uuid` | Yes | |
| `role_id` | `uuid` | Yes | Parent role |
| `action` | `string` | Yes | e.g. `read`, `write`, `delete`, `approve` |
| `resource_type` | `string` | Yes | e.g. `contract`, `invoice`, `drive_folder` |
| `conditions` | `json` | No | Phase 4+. Additional conditions (classification, department, etc.) |
| `effective_from` | `datetime` | Yes | |
| `effective_to` | `datetime \| null` | Yes | |

**Action naming**

- Lowercase snake_case recommended (`read`, `write`, `delete`, `approve`, `delegate`)
- Wildcard `*` to be considered from Phase 2 onward

---

### 5.7 Resource (Phase 4; schema only in Phase 1)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `uuid` | Yes | |
| `type` | `string` | Yes | e.g. `drive_folder`, `slack_channel`, `contract` |
| `external_id` | `string` | No | ID in external system |
| `owner_user_id` | `uuid \| null` | No | Owner |
| `owner_organization_id` | `uuid \| null` | No | Owning department |
| `classification` | `enum` | No | `public` \| `internal` \| `confidential` \| `restricted` |
| `effective_from` | `datetime` | Yes | Resource validity start |
| `effective_to` | `datetime \| null` | Yes | Resource validity end |

---

### 5.8 Assignment (Membership and Role Grant)

Relationship between user, organization, and role. Phase 1 represents `MEMBER_OF` + role grant via the Assignment API.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `uuid` | Yes | |
| `user_id` | `uuid` | Yes | |
| `organization_id` | `uuid` | Yes | |
| `role_id` | `uuid` | Yes | |
| `relationship` | `enum` | Yes | Integrated with Graph relationships from Phase 3 |
| `effective_from` | `datetime` | Yes | |
| `effective_to` | `datetime \| null` | Yes | |

---

## 6. Organization Graph (Phase 3)

Relationship types among organizations, users, and projects.

| Relationship | Subject | Object | Description | Example |
|--------------|---------|--------|-------------|---------|
| `BELONGS_TO` | User / Org | Organization | Membership | User → Sales Department |
| `REPORTS_TO` | User | User | Reporting line | Member → Manager |
| `MANAGES` | User | User / Org | Management authority | Manager → Team |
| `MEMBER_OF` | User | Organization | Member (equivalent to Assignment) | Project member |
| `OWNER_OF` | User / Org | Resource | Resource owner | Contract owner |
| `APPROVER_OF` | User | Resource / Org | Approver | Contract approver |
| `DELEGATED_TO` | User | User | Delegated authority | Acting approver during leave |
| `PROJECT_MEMBER` | User | Project | Project member | |
| `PROJECT_OWNER` | User | Project | Project owner | |

**Graph traversal rules (Phase 3+)**

- Only edges valid at the given `effective_at` are traversable during authorization
- If multiple paths exist, `allow = true` when any single path satisfies a Permission (OR evaluation)
- Circular references are rejected at registration time

---

## 7. Authorization Evaluation

### 7.1 Evaluation Flow

```
1. Receive user_id, resource_id, action, effective_at
2. Verify user is active at effective_at
3. Resolve user's organization memberships and roles on the Organization Graph at effective_at
4. Match action / resource_type against Permissions bound to resolved roles
5. Verify resource classification and validity period via Resource Graph (Phase 4)
6. Return (allow, reason, decision_id) and record in Audit Log
```

### 7.2 Decision Result

| Field | Type | Description |
|-------|------|-------------|
| `allow` | `boolean` | Allow / deny |
| `reason` | `string[]` | Human-readable reason chain (simplified Explain) |
| `decision_id` | `uuid` | Unique ID for audit and Explain |
| `deny_code` | `string` | Machine-readable deny code (e.g. `no_matching_permission`, `user_inactive`, `resource_expired`) |

### 7.3 Example

**Input**

- User: Sales Department Manager as of 2025-02-01
- Resource: 2024 Sales Contract (`type=contract`, `classification=confidential`)
- action: `read`
- effective_at: `2025-02-01T00:00:00Z`

**Expected output**

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

## 8. Explainable Authorization (Phase 5)

### 8.1 Decision Tree Structure

`/explain` returns the decision for a `decision_id` as a **tree**.

```
User (user_id, display_name)
  └─ Organization (org_id, name, effective_from..to)
       └─ Role (role_id, name)
            └─ Permission (permission_id, action, resource_type)
                 └─ Policy (conditions)          ← Phase 4+
                      └─ Resource (resource_id, type, classification)
                           └─ Decision: ALLOW | DENY
```

### 8.2 Response Schema (Overview)

```json
{
  "decision_id": "uuid",
  "allow": true,
  "tree": {
    "node_type": "user",
    "id": "uuid",
    "label": "Taro Yamada",
    "children": [
      {
        "node_type": "organization",
        "id": "uuid",
        "label": "Sales Department",
        "effective_from": "2024-04-01T00:00:00Z",
        "effective_to": "2025-03-31T23:59:59Z",
        "children": ["..."]
      }
    ]
  }
}
```

**Acceptance criteria**

- For any `decision_id`, the returned Tree matches the decision recorded in the Audit Log
- For DENY, the first node where denial occurred includes `deny_reason`

---

## 9. REST API

Base URL: `/api/v1`  
Authentication: `Authorization: Bearer <JWT>`  
Content-Type: `application/json`  
Datetime: ISO 8601 UTC (e.g. `2025-02-01T00:00:00Z`)

### 9.1 POST /authorize

Point-in-time authorization evaluation.

**Request**

```json
{
  "user_id": "uuid",
  "resource_id": "uuid",
  "action": "read",
  "effective_at": "2025-02-01T00:00:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user_id` | `uuid` | Yes | User to evaluate |
| `resource_id` | `uuid` | Yes | Target resource |
| `action` | `string` | Yes | Operation |
| `effective_at` | `datetime` | Yes | Evaluation reference time |

**Response `200`**

```json
{
  "allow": true,
  "decision_id": "uuid",
  "reason": ["Sales Department", "Manager", "Contract Reader"],
  "deny_code": null
}
```

**Errors**

| Status | Condition |
|--------|-----------|
| `400` | Missing required fields, invalid `effective_at` format |
| `404` | `user_id` or `resource_id` not found |
| `401` | Unauthenticated |

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
  "tree": {}
}
```

| Status | Condition |
|--------|-----------|
| `404` | `decision_id` not found in Audit Log |

---

### 9.3 CRUD Endpoints (Phase 1)

| Method | Path | Description | Phase |
|--------|------|-------------|-------|
| `POST` | `/users` | Create user | 1 |
| `GET` | `/users/{id}` | Get user | 1 |
| `POST` | `/organizations` | Create organization | 1 |
| `GET` | `/organizations/{id}` | Get organization | 1 |
| `POST` | `/roles` | Create role | 1 |
| `POST` | `/permissions` | Create permission | 1 |
| `POST` | `/assignments` | Grant membership and role | 1 |
| `POST` | `/resources` | Create resource | 4 |

**Common error response**

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

### 9.4 POST /users (Example)

**Request**

```json
{
  "email": "taro@example.com",
  "google_id": "1082...",
  "employee_number": "E001",
  "display_name": "Taro Yamada",
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

### 9.5 POST /assignments (Example)

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

## 10. AI APIs (MCP / SDK)

Interfaces for AI Agents to call directly. Implemented as wrappers over the REST API.

| Function / Tool | REST Mapping | Phase | Description |
|-----------------|--------------|-------|-------------|
| `authorize()` | `POST /authorize` | 1 | Authorization evaluation |
| `explainDecision()` | `POST /explain` | 5 | Decision explanation |
| `searchUser()` | `GET /users` (query) | 1 | User search |
| `searchOrganization()` | `GET /organizations` (query) | 1 | Organization search |
| `searchRoleHistory()` | `GET /assignments` (temporal filter) | 2 | Role history |
| `searchOrganizationHistory()` | Graph query | 3 | Membership history |
| `searchPermission()` | `GET /permissions` (query) | 1 | Permission search |
| `searchResource()` | `GET /resources` (query) | 4 | Resource search |

### 10.1 MCP Server

- OrgGraph **ships an MCP Server by default**
- Transport: stdio (local) / SSE (remote)
- Exposes the tools above as MCP Tools
- Authentication: injected via environment variables or OAuth Token at MCP connection time

### 10.2 SDK

| Language | Phase | Notes |
|----------|-------|-------|
| Go | 1 | Same as server implementation language |
| TypeScript | 2 | AI Agent / Node |
| Python | 2 | AI Agent |
| Java | 3 | Enterprise |

SDKs provide:

- Typed Request / Response
- Automatic retry (429 / 5xx)
- Date type conversion for `effective_at`

---

## 11. Audit

**All `/authorize` calls** are persisted to the audit log.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | = `decision_id` |
| `timestamp` | `datetime` | Evaluation execution time (server time) |
| `effective_at` | `datetime` | Evaluation reference time |
| `user_id` | `uuid` | |
| `resource_id` | `uuid` | |
| `action` | `string` | |
| `allow` | `boolean` | Decision result |
| `reason` | `string[]` | Reason chain |
| `deny_code` | `string \| null` | |
| `agent_id` | `string \| null` | Calling AI Agent identifier |
| `tool_name` | `string \| null` | Calling tool name (MCP tool name, etc.) |
| `requester` | `string` | API key / JWT `sub` / service account |

**Retention**: 7 years by default (configurable). Append-only for tamper resistance.

**Acceptance criteria**

- `/explain` content matches the Audit Log for the same `decision_id`
- `agent_id` / `tool_name` are required for MCP-originated calls

---

## 12. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| Availability | Phase 1: single region, 99.5% SLA (best-effort when self-hosted) |
| Latency | `/authorize` p99 < 200ms (no cache, graph depth ≤ 5) |
| Scalability | 100K users / 1M resources expected by Phase 4 |
| Security | TLS required, JWT validation, PII minimization in audit logs |
| Compatibility | REST API published as OpenAPI 3.1 |
| License | Apache-2.0 |

---

## 13. Phase 1 Acceptance Criteria (Summary)

- [ ] Login via Google Workspace OIDC
- [ ] CRUD for User / Organization / Role / Permission / Assignment
- [ ] `POST /authorize` accepts `effective_at` and returns `allow` / `reason` / `decision_id`
- [ ] All authorize calls are recorded in the Audit Log
- [ ] `effective_from` / `effective_to` exist on all major entities
- [ ] OpenAPI specification is published

---

## 14. Open Issues

| # | Topic | Options | Decision deadline |
|---|-------|---------|-------------------|
| 1 | Standard vocabulary for Permission `action` | CASL / custom enum | Before Phase 1 start |
| 2 | Role priority for concurrent posts | Union (OR) / explicit priority | Phase 3 |
| 3 | External resource ID sync method | Webhook / polling / MCP | Phase 6 |
| 4 | Graph DB selection | PostgreSQL + adjacency / Neo4j | Phase 3 design |
