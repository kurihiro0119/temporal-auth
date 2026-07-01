# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, etc.) when working with code in this repository.

## Project Overview

**OrgGraph** is a temporal authorization platform (not authentication) for the AI Agent era. It evaluates, at any specified point in time (`effective_at`), whether a user can perform an action on a resource given their organizational memberships, roles, and permissions at that moment. It integrates with external IdPs (Google Workspace, Entra ID, etc.) for identity but manages all authorization logic internally.

This repository is currently in the **planning/requirements phase** — no implementation exists yet.

## Documentation Conventions

This project maintains **paired Japanese and English documents**. When editing one, always update its counterpart in the same change:

| Japanese                                       | English                                     |
| ---------------------------------------------- | ------------------------------------------- |
| `docs/requirements/requirements-ja.md`         | `docs/requirements/requirements.md`         |
| `docs/requirements/product-requirements-ja.md` | `docs/requirements/product-requirements.md` |

Rules:

- Section structure, tables, API schemas, and acceptance criteria must match between language pairs.
- Keep code identifiers, JSON keys, HTTP paths, and enum values untranslated.
- Product name **OrgGraph** and phase names (**Phase 1**, **Phase 2**, etc.) are not translated.
- Each paired file starts with a language header linking to its counterpart.

## Domain Model

All major entities carry **temporal validity fields**:

- `effective_from`: datetime (ISO 8601 UTC) — validity start
- `effective_to`: datetime | null — validity end; `null` = currently active

Key entities: **User**, **Organization** (company/department/team/project), **Role**, **Permission** (action + resource_type bound to a role), **Resource**, **Assignment** (user ↔ organization ↔ role membership with a relationship type).

Authorization evaluation path: `user_id + resource_id + action + effective_at` → resolve memberships/roles on the org graph at `effective_at` → match permissions → return `{allow, reason[], decision_id, deny_code}`.

## Phased Architecture

| Phase | Scope |
| ----- | ----- |
| 1 | Google Workspace OIDC login; CRUD for User/Org/Role/Permission/Assignment; `POST /authorize`; Audit Log; Go SDK |
| 2 | Full temporal evaluation via `effective_at`; TypeScript + Python SDKs |
| 3 | Organization Graph (concurrent posts, delegation, approvers, graph traversal) |
| 4 | Resource Graph (temporal attributes, classification) |
| 5 | Explainable Authorization (`/explain` Decision Tree) |
| 6 | Google Workspace sync |
| 7 | SmartHR / Workday / Entra / Okta integration |
| 8 | AI Agent Runtime integration |

## API Design

- Base URL: `/api/v1`, Auth: `Authorization: Bearer <JWT>`, datetimes as ISO 8601 UTC
- Core endpoints: `POST /authorize`, `POST /explain`, plus CRUD endpoints for each entity
- Error shape: `{"error": {"code": "...", "message": "...", "details": []}}`
- OpenAPI 3.1 spec is a Phase 1 deliverable

AI Agent interfaces (MCP + SDK) are wrappers over the REST API. The MCP Server ships by default with stdio (local) and SSE (remote) transports.

## Implementation Notes (for when coding begins)

- The server implementation language is **Go** (Phase 1 SDK is Go)
- Graph DB technology is an open issue (PostgreSQL + adjacency list vs. Neo4j) — decision required before Phase 3
- Permission `action` naming convention (CASL vs. custom enum) is an open issue for Phase 1
- Audit log must be append-only, retained 7 years by default
- `/authorize` p99 latency target: < 200ms at graph depth ≤ 5
- License: Apache-2.0
