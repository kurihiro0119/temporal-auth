# Temporal Authorization Platform (仮称)

> **言語:** 日本語 · English: [product-requirements.md](./product-requirements.md)

## 概要

AI Agent時代における企業向け認可（Authorization）プラットフォームを提供する。

本プロダクトは認証(Authentication)を目的としない。

Google WorkspaceやMicrosoft Entra IDなどのIdentity Providerと連携し、
「誰が」「いつ」「どの組織で」「どの役職として」
「どのデータへアクセスできるか」
を管理する。

---

## 背景

従来の権限管理は、人がアプリケーションへログインすることを前提として設計されている。

しかしAI Agentは

- Google Drive
- Slack
- GitHub
- Notion
- Salesforce
- ERP

など複数システムを横断して業務を行う。

その際、

「現在のロール」

だけでは十分ではない。

例えば

2025年2月時点の営業資料

をAIが読みたい場合、

現在営業部に所属しているか

ではなく

2025年2月当時営業部だったか

を判断する必要がある。

つまり

時間軸(Time)

を持った認可が必要になる。

---

## 解決したい課題

### 1. Temporal Authorization

所属

役職

権限

すべてに有効期間を持たせる。

例

営業部
2024-04-01〜2025-03-31

AI推進室
2025-04-01〜

---

### 2. Organization Graph

企業は単純な部署構造ではない。

- 兼務
- PJ所属
- 承認者
- 代理
- 上司
- マネージャ

など多様な関係を持つ。

これらをGraphとして管理する。

---

### 3. Resource Graph

データにも時間が存在する。

例

契約書

有効期間

公開期間

機密区分

オーナー

部署

AIは

User Graph

Resource Graph

両方を見て認可を判断する。

---

### 4. Explainable Authorization

AIは

「なぜアクセスできるのか」

を説明できなければならない。

例

ALLOW

理由

営業部

↓

Manager

↓

契約書閲覧権限

↓

対象ドキュメント

---

### 5. AI Native

AI Agentが直接利用できるAPIを提供する。

MCP

REST API

SDK

を提供する。

---

## 対象外

以下は本OSSでは対象外とする。

- SSO
- MFA
- Password管理
- Identity Provider
- Email認証

これらはGoogle Workspaceなど既存サービスを利用する。

---

## ビジョン

AI Agentが企業で安全に働くための標準Authorization Platformになる。
