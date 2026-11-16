# SupplyMesh — Generated Diagrams Gallery

This directory contains high-resolution PNG exports of all architecture, workflow, database, and system design diagrams for **SupplyMesh: Global Procurement & Supplier Intelligence Platform**.

---

## 📋 Deliverable 01: Product Feature Design

| Diagram Name | File | Description |
|--------------|------|-------------|
| **Role Hierarchy** | [`01-01-role-hierarchy.png`](./01-01-role-hierarchy.png) | Organization Owner, Procurement Manager, Department Manager, Finance Officer, Employee hierarchy |
| **Workflow States** | [`01-02-workflow-states.png`](./01-02-workflow-states.png) | Purchase request lifecycle state machine (Draft → Submitted → Under Review → Approved/Rejected) |
| **User Journey 1: Supplier Onboarding** | [`01-03-journey-supplier-onboarding.png`](./01-03-journey-supplier-onboarding.png) | Step-by-step registration, document verification, and activation |
| **User Journey 2: PR & Multi-Level Approval** | [`01-04-journey-purchase-request-approval.png`](./01-04-journey-purchase-request-approval.png) | Tiered threshold evaluation, approval routing, escalation, and outcome notifications |
| **User Journey 3: PO & Invoice Processing** | [`01-05-journey-po-invoice-processing.png`](./01-05-journey-po-invoice-processing.png) | PO creation, vendor fulfillment, 3-way invoice matching, and payment release |

---

## 🗄️ Deliverable 02: Database Design

| Diagram Name | File | Description |
|--------------|------|-------------|
| **Entity Relationship Diagram (ERD)** | [`02-01-entity-relationship-diagram.png`](./02-01-entity-relationship-diagram.png) | Full schema of all 16 core entities: Organizations, Departments, Users, Roles, Permissions, Suppliers, Documents, PRs, Workflow Instances, Steps, POs, Items, Invoices, Payments, Performance, Audit Logs |

---

## 🏛️ Deliverable 03: Application Architecture

| Diagram Name | File | Description |
|--------------|------|-------------|
| **High-Level Modular Monolith Architecture** | [`03-01-high-level-architecture.png`](./03-01-high-level-architecture.png) | Client, NGINX Gateway, Next.js Frontend, NestJS Modular Monolith, PostgreSQL Primary/Replica, Redis, S3, BullMQ |
| **Authentication Flow (JWT + Redis)** | [`03-02-authentication-flow.png`](./03-02-authentication-flow.png) | Credential validation, JWT creation, Redis refresh token storage, and session cookie lifecycle |
| **Notification System Architecture** | [`03-03-notification-system.png`](./03-03-notification-system.png) | Event triggers, in-app notification DB persistence, BullMQ email queue, and worker dispatch |
| **Production Deployment Architecture** | [`03-04-deployment-architecture.png`](./03-04-deployment-architecture.png) | GitHub Actions CI/CD, AWS ECR, VPS Docker Compose (NGINX, Next.js, NestJS, BullMQ, Redis) + AWS RDS & S3 |

---

## ⚙️ Deliverable 04: System Design

| Diagram Name | File | Description |
|--------------|------|-------------|
| **Workflow 1: Supplier Registration & Verification** | [`04-01-supplier-registration-verification.png`](./04-01-supplier-registration-verification.png) | Pre-signed S3 document upload, validation, ClamAV checks, and admin verification sequence |
| **Workflow 2: PR Approval State Machine** | [`04-02-purchase-request-state-machine.png`](./04-02-purchase-request-state-machine.png) | Multi-step approval state progression and fallback paths |
| **Workflow 2: Detailed Approval Sequence** | [`04-03-purchase-request-approval-flow.png`](./04-03-purchase-request-approval-flow.png) | End-to-end request submission, workflow engine rule resolution, step assignments, atomic DB commits, notifications |
| **Workflow 3: PO Generation Sequence** | [`04-04-purchase-order-generation-flow.png`](./04-04-purchase-order-generation-flow.png) | PR conversion, supplier validation, sequential PO number generation, line items insertion, PDF dispatch |
| **Workflow 4: Invoice Matching & Payment Flow** | [`04-05-invoice-verification-payment-flow.png`](./04-05-invoice-verification-payment-flow.png) | 3-way automated matching engine, discrepancy detection, approval override, and overpayment prevention |
| **System Scalability Roadmap** | [`04-06-system-scalability-roadmap.png`](./04-06-system-scalability-roadmap.png) | Phase 1 (MVP Docker Compose) → Phase 2 (AWS RDS/Redis) → Phase 3 (ECS Auto-Scale) → Phase 4 (Tenant Sharding) |
