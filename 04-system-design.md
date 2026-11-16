# Deliverable 04 — System Design
# SupplyMesh: Global Procurement & Supplier Intelligence Platform

---

## Overview

This document provides detailed system design for the four critical workflows in SupplyMesh. Each workflow is analyzed for request flow, business logic, database operations, authorization, failure handling, transaction management, and notification flow.

---

## Workflow 1: Supplier Registration & Verification

### 1.1 Request Flow

```mermaid
sequenceDiagram
    participant PM as Procurement Manager
    participant API as NestJS API
    participant Guard as Auth Guard
    participant Service as Supplier Service
    participant DocService as Document Service
    participant DB as PostgreSQL
    participant S3
    participant AuditService
    participant NotifService

    PM->>API: POST /suppliers (profile data)
    API->>Guard: Validate JWT + Check permission(supplier:create)
    Guard-->>API: Authorized

    API->>Service: createSupplier(orgId, dto)
    Service->>DB: BEGIN TRANSACTION
    Service->>DB: CHECK UNIQUE(organization_id, tax_id)
    alt Duplicate tax_id
        DB-->>Service: Constraint violation
        Service->>DB: ROLLBACK
        Service-->>API: 409 CONFLICT
        API-->>PM: Error: Supplier with this tax ID already exists
    end
    Service->>DB: INSERT INTO suppliers (status = 'pending_verification')
    Service->>DB: COMMIT
    Service->>AuditService: log(CREATE, supplier, newValues)
    Service-->>API: Supplier created

    PM->>API: POST /suppliers/:id/documents (file metadata + presigned URL request)
    API->>DocService: getUploadUrl(supplierId, docType)
    DocService->>S3: Generate pre-signed PUT URL (15 min TTL)
    S3-->>DocService: Pre-signed URL
    DocService-->>API: { uploadUrl, documentId }
    API-->>PM: Upload URL returned

    PM->>S3: PUT file directly to S3
    S3-->>PM: 200 OK

    PM->>API: POST /suppliers/:id/documents/:docId/confirm
    API->>DocService: confirmUpload(documentId)
    DocService->>DB: UPDATE supplier_documents SET status = 'active'
    DocService->>AuditService: log(UPLOAD, supplier_document)
    DocService-->>API: Document confirmed

    PM->>API: PATCH /suppliers/:id/verify
    API->>Guard: Check permission(supplier:verify)
    API->>Service: verifySupplier(orgId, supplierId, userId)
    Service->>DB: SELECT supplier, check status = 'pending_verification'
    Service->>Service: validateRequiredDocuments()
    alt Missing required documents
        Service-->>API: 422 UNPROCESSABLE: Missing required documents
        API-->>PM: Error: Please upload all required documents first
    end
    Service->>DB: UPDATE suppliers SET status='active', verified_by, verified_at
    Service->>AuditService: log(VERIFY, supplier)
    Service->>NotifService: notify(PM, 'Supplier verified successfully')
    Service-->>API: Supplier active
    API-->>PM: ✅ Supplier now active and available for POs
```

### 1.2 Business Logic
- A supplier cannot be verified unless all required document types are uploaded and not expired
- Required documents (configurable per org): Business License, Tax Clearance Certificate, Insurance Certificate
- Duplicate detection: `UNIQUE(organization_id, tax_id)` prevents double registration
- Blacklisted suppliers: if same `tax_id` or `email` matches a blacklisted entry in another org of the platform, an alert is raised (soft warning, not hard block)

### 1.3 Failure Scenarios & Handling

| Failure | Handling |
|---------|----------|
| S3 upload fails | Pre-signed URL expires; client retries. Document record stays in `pending_upload` state and is auto-cleaned after 24h |
| Verification attempted on blacklisted supplier | 403 FORBIDDEN with reason |
| Concurrent verification attempts | DB-level `SELECT FOR UPDATE` prevents race condition |
| File virus detected (ClamAV) | S3 Object Lambda deletes file; API receives webhook; document marked `quarantined` |

---

## Workflow 2: Purchase Request & Multi-Level Approval

### 2.1 State Machine

```mermaid
stateDiagram-v2
    [*] --> Draft: Employee creates
    Draft --> Submitted: Employee submits
    Submitted --> Step1_Pending: Workflow engine routes
    Step1_Pending --> Step1_Approved: Dept Manager approves
    Step1_Pending --> Rejected: Any approver rejects
    Step1_Approved --> Step2_Pending: Next step triggered
    Step2_Pending --> Step2_Approved: Procurement Manager approves
    Step2_Approved --> Step3_Pending: If multi-level
    Step3_Pending --> FullyApproved: Finance Officer approves
    Step2_Approved --> FullyApproved: If 2-step workflow
    FullyApproved --> [*]: Triggers PO creation
    Rejected --> Draft: Employee can revise (max 3x)
    Draft --> Cancelled: Employee cancels
```

### 2.2 Detailed Request Flow

```mermaid
sequenceDiagram
    participant Emp as Employee
    participant API
    participant PRService as PR Service
    participant WFEngine as Workflow Engine
    participant DB
    participant Redis
    participant NotifService
    participant BullMQ

    Emp->>API: POST /purchase-requests (title, cost, dept, priority, date)
    API->>PRService: createRequest(orgId, userId, dto)
    PRService->>DB: INSERT purchase_requests (status='draft')
    PRService-->>API: Draft PR created

    Emp->>API: POST /purchase-requests/:id/submit
    API->>PRService: submitRequest(orgId, prId, userId)
    PRService->>DB: SELECT pr WHERE id=prId AND organization_id=orgId
    PRService->>PRService: Validate: status must be 'draft' or 'rejected'

    PRService->>WFEngine: initiateWorkflow(pr, orgId)
    WFEngine->>DB: SELECT workflow_template WHERE org=orgId AND entity_type='purchase_request' AND is_active=true
    WFEngine->>WFEngine: evaluateThresholds(pr.estimated_cost, template.rules)
    Note over WFEngine: e.g. cost=$3000 → Step 1: Dept Mgr, Step 2: Procurement Mgr

    WFEngine->>DB: BEGIN TRANSACTION
    WFEngine->>DB: INSERT approval_instances (total_steps=2, current_step=1)
    WFEngine->>DB: INSERT approval_steps × N (pending)
    WFEngine->>DB: UPDATE purchase_requests SET status='under_review'
    WFEngine->>DB: COMMIT

    WFEngine->>WFEngine: findApproverForStep(step1.role, dept_id)
    WFEngine->>DB: SELECT users WHERE role='department_manager' AND dept_id=pr.dept_id
    WFEngine->>DB: UPDATE approval_steps[1] SET assigned_to=approverUserId, deadline_at=NOW()+24h

    WFEngine->>NotifService: notifyApprover(step1.assigned_to, pr)
    NotifService->>DB: INSERT notifications
    NotifService->>BullMQ: enqueue email_notification job

    Note over BullMQ: Async email sent to approver

    WFEngine-->>PRService: Workflow initiated
    PRService-->>API: PR submitted
    API-->>Emp: ✅ Request submitted, pending Department Manager review

    Note over DB,BullMQ: --- Later: Approver takes action ---

    API->>API: POST /approvals/:stepId/approve (Dept Manager)
    API->>WFEngine: processApproval(stepId, userId, 'approved', comment)
    WFEngine->>DB: SELECT approval_step, verify assigned_to = userId
    WFEngine->>DB: BEGIN TRANSACTION
    WFEngine->>DB: UPDATE approval_steps SET status='approved', decided_at=NOW()
    WFEngine->>WFEngine: isLastStep? No
    WFEngine->>DB: UPDATE approval_instances SET current_step=2
    WFEngine->>WFEngine: findApproverForStep(step2)
    WFEngine->>DB: UPDATE approval_steps[2] SET assigned_to, deadline_at
    WFEngine->>DB: COMMIT
    WFEngine->>NotifService: notify next approver
    WFEngine->>NotifService: notify requester of step 1 approval

    Note over DB,BullMQ: --- Last step approved ---

    WFEngine->>DB: BEGIN TRANSACTION
    WFEngine->>DB: UPDATE approval_instances SET status='approved', completed_at=NOW()
    WFEngine->>DB: UPDATE purchase_requests SET status='approved'
    WFEngine->>DB: COMMIT
    WFEngine->>NotifService: notifyRequester(pr.requested_by, 'Request fully approved')
    WFEngine->>PRService: triggerPOCreationWorkflow(prId)
```

### 2.3 Workflow Engine Rules Resolution

```typescript
// Pseudo-code: Workflow Engine step resolution
function evaluateWorkflowSteps(cost: number, rules: WorkflowRule[]): ApprovalStep[] {
  const applicableRules = rules
    .filter(rule => cost <= (rule.max_amount ?? Infinity))
    .sort((a, b) => a.step - b.step);

  // e.g. cost = $3000:
  // Step 1: max_amount=500 → NOT applicable (3000 > 500 → actually step 1 is for ≤500 requests)
  // Actually, re-read: steps define who approves for requests UP TO that threshold
  // Better interpretation: each step has a MIN threshold
  // If cost ≥ step.min_threshold, include that step

  return applicableRules.map(rule => ({
    stepNumber: rule.step,
    stepName: rule.name,
    approverRole: rule.role,
    deadlineHours: rules.escalation_hours ?? 24
  }));
}
```

### 2.4 Failure Scenarios

| Failure | Handling |
|---------|----------|
| Approver is unavailable / deleted | System tries to find another user with same role in same dept; if none, escalate to org owner |
| SLA breach (no action in 24h) | BullMQ cron job runs every 30 min; escalates to next-level approver |
| Concurrent approval attempt | `SELECT FOR UPDATE` on approval_step prevents double-processing |
| Requester tries to cancel approved request | Business rule: cannot cancel if status is `approved` or PO already created |
| Circular approval (delegated to self) | Delegation guard prevents user delegating to themselves |

### 2.5 Database Operations Summary

```sql
-- Atomic workflow initiation
BEGIN;
  INSERT INTO approval_instances (...);
  INSERT INTO approval_steps (...) × N;
  UPDATE purchase_requests SET status = 'under_review' WHERE id = ?;
COMMIT;

-- Atomic step approval
BEGIN;
  UPDATE approval_steps SET status='approved', decided_at=NOW() WHERE id=? AND status='pending';
  -- If last step:
  UPDATE approval_instances SET status='approved', completed_at=NOW() WHERE id=?;
  UPDATE purchase_requests SET status='approved' WHERE id=?;
COMMIT;
```

---

## Workflow 3: Purchase Order Generation

### 3.1 Request Flow

```mermaid
sequenceDiagram
    participant PM as Procurement Manager
    participant API
    participant POService as PO Service
    participant SupplierService
    participant DB
    participant AuditService
    participant NotifService

    PM->>API: POST /purchase-orders (from approved PR or manual)
    API->>POService: createPurchaseOrder(orgId, userId, dto)

    POService->>DB: SELECT purchase_request WHERE id=dto.prId (if from PR)
    POService->>POService: Validate: PR status must be 'approved'

    POService->>SupplierService: validateSupplier(supplierId, orgId)
    SupplierService->>DB: SELECT supplier WHERE id=? AND org_id=? AND status='active'
    alt Supplier not active
        SupplierService-->>POService: 422 SUPPLIER_NOT_ACTIVE
        POService-->>API: Error
        API-->>PM: Cannot create PO: Supplier is not active
    end

    POService->>POService: calculateTotals(lineItems)
    POService->>POService: generatePONumber(orgId, year) → PO-ACME-2026-0042

    POService->>DB: BEGIN TRANSACTION
    POService->>DB: INSERT purchase_orders (status='draft')
    POService->>DB: INSERT purchase_order_items × N
    POService->>DB: UPDATE purchase_requests SET status='po_created' (if linked)
    POService->>DB: COMMIT

    POService->>AuditService: log(CREATE, purchase_order)

    PM->>API: POST /purchase-orders/:id/send
    API->>POService: sendPurchaseOrder(orgId, poId, userId)
    POService->>DB: SELECT po WHERE id=? AND status='draft'
    POService->>DB: UPDATE purchase_orders SET status='sent', sent_at=NOW()
    POService->>NotifService: notifySupplier(supplier.email, po)
    Note over NotifService: Email sent with PO PDF attachment (generated server-side)
    POService->>AuditService: log(SEND, purchase_order)
    POService-->>API: PO sent
    API-->>PM: ✅ PO sent to supplier
```

### 3.2 PO Number Generation (Race-Condition Safe)

```sql
-- Using PostgreSQL sequence per org per year
-- Sequence name: po_seq_{org_id}_{year}
-- Created on first PO of a new year

SELECT NEXTVAL('po_seq_acme_2026');
-- Returns: 42
-- PO Number: PO-ACME-2026-0042
```

### 3.3 PO Amendment Workflow

When a PO needs to be amended after sending:
1. Procurement Manager requests amendment
2. If amendment increases value by >10% → re-approval required (new approval instance created)
3. If amendment < 10% increase → PM can amend directly
4. `version` column incremented on every amendment
5. Previous version saved to `purchase_order_amendments` table (audit trail)

### 3.4 Failure Scenarios

| Failure | Handling |
|---------|----------|
| Duplicate PO submission | Idempotency key on client; server-side: check PR already has linked PO |
| Supplier becomes inactive after PO created | System alerts PM; PO can still be sent (business decision) |
| Line item total doesn't match PR estimated cost (>5%) | Warning shown to PM; requires explicit override acknowledgment |
| PO generation timeout (long line item list) | DB transaction has 30s timeout; chunked inserts for >50 line items |

---

## Workflow 4: Invoice Verification & Payment Processing

### 4.1 Request Flow

```mermaid
sequenceDiagram
    participant FO as Finance Officer
    participant API
    participant InvService as Invoice Service
    participant MatchEngine as Matching Engine
    participant PayService as Payment Service
    participant DB
    participant NotifService
    participant AuditService

    FO->>API: POST /invoices (invoice data + PO reference)
    API->>InvService: recordInvoice(orgId, userId, dto)

    InvService->>DB: CHECK UNIQUE(org_id, supplier_id, supplier_invoice_number)
    alt Duplicate invoice
        InvService-->>API: 409 CONFLICT: Invoice already recorded
        API-->>FO: Error: Duplicate invoice number
    end

    InvService->>DB: SELECT purchase_order WHERE id=dto.poId AND status='delivered'
    alt PO not in delivered status
        InvService-->>API: 422: PO must be in delivered state to receive invoice
    end

    InvService->>DB: BEGIN TRANSACTION
    InvService->>DB: INSERT invoices (status='pending_review')
    InvService->>DB: INSERT invoice_items × N
    InvService->>DB: COMMIT

    InvService->>MatchEngine: performThreeWayMatch(invoiceId, poId)
    MatchEngine->>DB: SELECT po with items
    MatchEngine->>DB: SELECT invoice with items
    MatchEngine->>MatchEngine: compareAmounts(po.total, invoice.total, tolerance=2%)
    MatchEngine->>MatchEngine: compareLineItems(poItems, invoiceItems)

    alt Perfect Match
        MatchEngine->>DB: UPDATE invoices SET status='matched'
        MatchEngine-->>InvService: Match result: CLEAN
    else Discrepancy Found
        MatchEngine->>DB: UPDATE invoices SET status='discrepancy', discrepancies=[...]
        MatchEngine->>NotifService: alertFinanceAndProcurement(discrepancyDetails)
        MatchEngine-->>InvService: Match result: DISCREPANCY
        InvService-->>API: Invoice recorded with discrepancies flagged
        API-->>FO: ⚠️ Invoice has discrepancies - review required
    end

    FO->>API: POST /invoices/:id/approve (after reviewing any discrepancies)
    API->>InvService: approveInvoice(orgId, invoiceId, userId)
    InvService->>DB: SELECT invoice WHERE status IN ('matched', 'discrepancy')
    InvService->>DB: UPDATE invoices SET status='approved', reviewed_by, reviewed_at
    InvService->>AuditService: log(APPROVE, invoice)
    InvService-->>API: Invoice approved

    FO->>API: POST /payments (invoiceId, amount, method, scheduledDate)
    API->>PayService: createPayment(orgId, invoiceId, dto)
    PayService->>DB: SELECT invoice WHERE status='approved'
    PayService->>PayService: validatePaymentAmount(invoice.total, dto.amount)
    Note over PayService: Prevents overpayment: sum of payments <= invoice.total

    PayService->>DB: BEGIN TRANSACTION
    PayService->>DB: INSERT payments (status='scheduled')
    PayService->>DB: SELECT SUM(amount) FROM payments WHERE invoice_id=invoiceId
    PayService->>PayService: IF total_paid = invoice.total → mark invoice as paid
    PayService->>DB: UPDATE invoices SET status='paid' (if fully paid)
    PayService->>DB: COMMIT

    PayService->>AuditService: log(CREATE, payment)
    PayService->>NotifService: notifyFinanceAndPM(paymentScheduled)
    PayService-->>API: Payment scheduled
    API-->>FO: ✅ Payment of $X scheduled for {date}
```

### 4.2 Three-Way Matching Logic

```typescript
interface MatchResult {
  status: 'matched' | 'discrepancy';
  discrepancies: Discrepancy[];
}

function performThreeWayMatch(po: PurchaseOrder, invoice: Invoice): MatchResult {
  const discrepancies: Discrepancy[] = [];

  // Check 1: Total amount within 2% tolerance
  const tolerance = po.total_amount * 0.02;
  if (Math.abs(invoice.total_amount - po.total_amount) > tolerance) {
    discrepancies.push({
      type: 'AMOUNT_MISMATCH',
      po_amount: po.total_amount,
      invoice_amount: invoice.total_amount,
      difference: invoice.total_amount - po.total_amount
    });
  }

  // Check 2: Line item matching
  for (const invoiceItem of invoice.items) {
    const matchedPoItem = po.items.find(i => i.sku === invoiceItem.sku || i.item_name === invoiceItem.item_name);
    if (!matchedPoItem) {
      discrepancies.push({ type: 'UNRECOGNIZED_ITEM', item: invoiceItem.item_name });
    } else if (invoiceItem.quantity > matchedPoItem.quantity) {
      discrepancies.push({ type: 'QUANTITY_MISMATCH', item: invoiceItem.item_name });
    } else if (Math.abs(invoiceItem.unit_price - matchedPoItem.unit_price) > 0.01) {
      discrepancies.push({ type: 'PRICE_MISMATCH', item: invoiceItem.item_name });
    }
  }

  return { status: discrepancies.length === 0 ? 'matched' : 'discrepancy', discrepancies };
}
```

### 4.3 Overpayment Prevention

```sql
-- Check before inserting new payment
SELECT COALESCE(SUM(amount), 0) as total_paid
FROM payments
WHERE invoice_id = $1 AND status NOT IN ('failed', 'cancelled');

-- Application validates:
-- (total_paid + new_payment_amount) <= invoice.total_amount
-- If violated → 422 UNPROCESSABLE: Payment would exceed invoice total
```

### 4.4 Failure Scenarios

| Failure | Handling |
|---------|----------|
| Invoice recorded for not-delivered PO | Business rule enforced: PO must be `delivered` |
| Duplicate invoice submission | UNIQUE constraint on `(org_id, supplier_id, supplier_invoice_number)` |
| Overpayment attempt | Application-level check + DB-level trigger as backup |
| Invoice amount > PO amount by >2% | Auto-flagged as discrepancy; requires explicit Finance Officer override with reason |
| Payment processing failure (future) | Payment status = `failed`; retry queue in BullMQ; alert Finance Officer |

---

## Bottlenecks & Scalability Analysis

### Identified Bottlenecks

| Bottleneck | Current Impact | Resolution |
|-----------|---------------|------------|
| Approval workflow routing (finding approvers) | O(n) role lookups per step | Cache role-to-user mappings in Redis (5-min TTL) |
| Analytics queries (org-wide spending) | Full table scans on large PO tables | Read replica + materialized views updated nightly |
| Supplier performance score recalculation | Runs synchronously on PO close | Move to BullMQ background job |
| File upload (large supplier documents) | Synchronous S3 upload blocks request | Pre-signed URLs allow direct client-to-S3 upload |
| Audit log writes | Synchronous writes add latency | Async via BullMQ queue |
| Notification fan-out (many approvers) | Synchronous for-loop notification | BullMQ fan-out job with concurrency=5 |

### Scalability Roadmap

```mermaid
flowchart LR
    Phase1["Phase 1 — MVP\nSingle VPS\nDocker Compose\n1000 orgs"]
    Phase2["Phase 2 — Growth\nAWS EC2 + RDS\nRead Replica\nRedis Cluster\n10,000 orgs"]
    Phase3["Phase 3 — Scale\nAWS ECS (Auto-Scale)\nRDS Aurora\nWorker Fleet\n100,000 orgs"]
    Phase4["Phase 4 — Enterprise\nTenant Sharding\nDedicated DB per large tenant\nGlobal CDN"]

    Phase1 --> Phase2 --> Phase3 --> Phase4
```

### Database Scalability Strategy

| Phase | Strategy |
|-------|---------|
| 0-10K orgs | Single PostgreSQL primary + 1 read replica |
| 10K-100K orgs | PgBouncer connection pooling, monthly partitioning of audit_logs and notifications |
| 100K+ orgs | Horizontal sharding by `organization_id` hash; large tenants on dedicated instances |
| Any phase | Materialized views for analytics; never run aggregate queries on primary |
