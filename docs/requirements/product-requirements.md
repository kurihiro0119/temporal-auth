# Temporal Authorization Platform (Working Title)

> **Language:** English · Japanese: [product-requirements-ja.md](./product-requirements-ja.md)

## Overview

This product provides an enterprise authorization platform for the AI Agent era.

It is not intended to provide authentication.

Instead, it integrates with identity providers such as Google Workspace and Microsoft Entra ID, and manages:

- who the user is
- when access is evaluated
- which organization the user belongs to
- which role the user has
- which data the user can access

---

## Background

Traditional access control is designed around humans logging in to applications.

AI Agents, however, operate across multiple systems such as:

- Google Drive
- Slack
- GitHub
- Notion
- Salesforce
- ERP

In that context, the user's current role is not enough.

For example, if an AI Agent needs to read sales materials from February 2025, the platform should not only check whether the user currently belongs to the Sales department.

It must check whether the user belonged to the Sales department at that point in time.

In other words, authorization needs a time dimension.

---

## Problems To Solve

### 1. Temporal Authorization

The platform gives validity periods to:

- organizational memberships
- roles
- permissions

Example:

Sales Department
2024-04-01 to 2025-03-31

AI Promotion Office
2025-04-01 onward

---

### 2. Organization Graph

Enterprise organizations are not simple department trees.

They include many relationship types, such as:

- concurrent posts
- project memberships
- approvers
- delegates
- managers
- supervisors

These relationships are managed as a graph.

---

### 3. Resource Graph

Data also has time-based properties.

Example:

Contract document

- validity period
- publication period
- confidentiality classification
- owner
- department

An AI Agent evaluates authorization using both the User Graph and the Resource Graph.

---

### 4. Explainable Authorization

The platform must be able to explain why access is allowed or denied.

Example:

ALLOW

Reason:

Sales Department

↓

Manager

↓

Contract document read permission

↓

Target document

---

### 5. AI Native

The platform provides APIs that AI Agents can use directly.

It provides:

- MCP
- REST API
- SDK

---

## Out Of Scope

The following are out of scope for this OSS project:

- SSO
- MFA
- password management
- identity providers
- email authentication

These should be handled by existing services such as Google Workspace.

---

## Vision

To become the standard authorization platform that allows AI Agents to work safely inside enterprises.
