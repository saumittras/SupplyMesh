<div align="center">

# 🌐 SupplyMesh
### Global Procurement & Supplier Intelligence Platform

[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Next.js-15-black.svg)](https://next.js5.org/)
[![NestJS](https://img.shields.io/badge/NestJS-Modular-red.svg)](https://nestjs.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blueviolet.svg)](https://www.postgresql.org/)
[![Drizzle ORM](https://img.shields.io/badge/Drizzle-ORM-C5F74F.svg)](https://orm.drizzle.team/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

*An enterprise-grade, multi-tenant Procure-to-Pay (P2P) SaaS platform designed for medium-to-large organizations to manage suppliers, multi-level purchase approvals, automated PO conversion, 3-way invoice matching, and supplier risk intelligence.*

</div>

---

## 🚀 Core Features

* **Strict Multi-Tenant Architecture:** Logical tenant isolation enforced at both the NestJS application layer and PostgreSQL Row-Level Security (RLS) level.
* **Supplier Lifecycle Management:** Comprehensive onboarding, KYC/tax document verification, and performance risk scoring.
* **Dynamic Multi-Level Approval Engine:** Configurable rule-based approval workflows driven by monetary thresholds (e.g., `<$500`, `$500–$5,000`, `>$5,000`), department rules, and role hierarchies.
* **Purchase Order & Inventory Tracking:** Automated conversion of approved purchase requests into formal purchase orders with line-item granularity and shipment tracking.
* **3-Way Invoice Matching:** Automated discrepancy detection comparing Purchase Requests, Purchase Orders, and Supplier Invoices before payment authorization.
* **Supplier Intelligence Scorecards:** Automated metric tracking for on-time delivery rates, order completion rates, and historical spending risk indicators.
* **Immutable Audit Trail:** Comprehensive append-only audit logging for every state mutation across organizations.

---

## 🛠️ Technology Stack

* **Frontend:** Next.js 15 (App Router), React, TypeScript, Tailwind CSS, Shadcn UI
* **Backend:** Node.js, NestJS (Modular Monolith Architecture)
* **Database & ORM:** PostgreSQL 16, Drizzle ORM
* **Caching & Job Queue:** Redis, BullMQ (for async email alerts & performance calculations)
* **Storage:** AWS S3 (Encrypted Supplier KYC & Invoice PDFs)
* **DevOps & Deployment:** Docker, Multi-Container Architecture, NGINX, GitHub Actions CI/CD, AWS / VPS

---

## 📂 Repository Structure

```text
SupplyMesh/
├── .ai-context/                # AI agent guardrails and domain rules
├── apps/
│   ├── web/                    # Next.js 15 Frontend Client Portal
│   └── api/                    # NestJS Modular Monolith Backend Core
├── packages/
│   ├── database/               # Drizzle ORM schemas, migrations & seed scripts
│   └── shared-types/           # Shared TypeScript interfaces & DTOs
├── docker-compose.yml          # Local development stack (Postgres + Redis)
└── README.md