# Architecture Requirements Specification v2

## Author
- Author: Analysis/Design Agent
- Date: 2026-04-10
- Version: v2.0

## Purpose
Define implementable functional, interface, data, and operational requirements for OMS, WMS, and Master Data Hub aligned with architecture boundaries.

## Audience
Developers, solution architects, QA engineers, and project managers.

## Table of Contents
- 1. Scope and Principles
- 2. Functional Requirements
- 3. Non-Functional Requirements
- 4. Interface, Data, and Process Linkage
- 5. Operations
- 6. Definition of Done

## Main Content

### 1. Scope and Principles
- Scope: OMS, WMS, Master Data Hub, and internal integration APIs.
- Out of scope: external factory implementation (only extension points are defined).
- Design principles:
  - Clear service boundaries: OMS (order decision), WMS (logistics execution), MDH (master data).
  - Hybrid synchronous API and asynchronous event model.
  - Single source of truth for master data.

### 2. Functional Requirements

#### 2.1 OMS
- Order intake:
  - Supported channels: `B2B_PORTAL`, `MARKETPLACE`, `EDI`, `MANUAL`.
  - Input formats: JSON (REST), CSV (batch upload), EDI X12 via adapter.
  - Required fields: `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items[].skuCode`, `items[].qty`, `items[].uom`.
  - Validation: uniqueness by `channel + externalOrderNo`, `qty > 0`, ISO-8601 datetime, MDH existence check, idempotency key.
  - Output: success returns internal order number with `RECEIVED`; failure returns field-level error.
- Routing:
  - Inputs: destination region, available inventory, center cutoff, SLA grade.
  - Priority: SLA compliance -> inventory fill rate -> operating cost -> fixed tie-breaker.
  - Outputs: center assignment or split plan with routing reason code.
- Priority scoring:
  - Formula: `priorityScore = (slaWeight*slaUrgency) + (vipWeight*customerTier) + (delayWeight*agingHours) + (penaltyWeight*stockRisk)`.
  - Defaults: `slaWeight=0.4`, `vipWeight=0.2`, `delayWeight=0.3`, `penaltyWeight=0.1`.
  - Bands: `CRITICAL(>=80)`, `HIGH(60-79)`, `NORMAL(40-59)`, `LOW(<40)`.

#### 2.2 WMS
- Inbound flow: `ASN_RECEIVED -> ARRIVED -> INSPECTED -> PUTAWAY_DONE -> RECEIPT_CONFIRMED`.
- Outbound flow: `ALLOCATED -> PICKING -> PACKED -> SHIPPED -> HANDED_OVER`.
- Rules:
  - Single wave per order in center by default.
  - High urgency orders are queued to immediate picking.
- Inventory API and cache:
  - Metrics: `onHand`, `available`, `allocated`, `inTransit`.
  - SLA: 95p <= 300ms.
  - Cache TTL 3 seconds, immediate invalidation on inventory events.

#### 2.3 Master Data Hub
- Core domains: Product, Customer, FulfillmentCenter, RoutingRule.
- Validation:
  - Code keys allow `A-Z`, `0-9`, `-`.
  - `effectiveFrom <= effectiveTo`.
  - Enum status: `ACTIVE`, `INACTIVE`, `DEPRECATED`.
- Synchronization:
  - One-way distribution MDH -> OMS/WMS.
  - Real-time change events + daily full snapshot.
  - Latest version wins; schema mismatch is rejected with alert.

### 3. Non-Functional Requirements
- Performance:
  - Order intake API 95p <= 500ms.
  - Inventory query API 95p <= 300ms.
  - Throughput >= 50,000 order lines/hour.
  - Concurrency: 300 back-office users, 2,000 API connections.
- Availability: SLA 99.9%, RTO <= 30 minutes, RPO <= 5 minutes, Active-Standby failover.
- Security: OAuth2 client credential + JWT, RBAC, 1-year audit retention, TLS 1.2+, AES-256 at rest.
- Scalability: horizontal API scale-out, rule hot update, center onboarding by master data registration.

### 4. Interface, Data, and Process Linkage
- API contract: `arch_api_contract_v1.yaml`
- Data model and ERD: `arch_data_model_v1.md`
- Fulfillment process and exception handling: `arch_order_fulfillment_flow_v1.md`

### 5. Operations
- Startup:
  - Verify MDH snapshot load.
  - Verify OMS/WMS health checks.
  - Open intake after queue backlog equals zero.
- Shutdown:
  - Block new intake.
  - Drain in-flight orders.
  - Flush queues before termination.
- Incident response:
  - Severity model: Sev1-Sev3.
  - Sev1 actions: on-call in 5 min, workaround/rollback in 15 min, customer notice in 30 min.
- Monitoring thresholds:
  - Routing failure > 1% triggers alert.
  - Queue delay > 5 minutes triggers alert.
- Backup and DR:
  - Incremental backup every 5 minutes.
  - Full backup daily.
  - DR rehearsal monthly.

### 6. Definition of Done
- Development team can implement API, data, and process from this document set.
- Architecture boundaries do not conflict with WJA-10 system decomposition.
- PM, analysis/design, and customer checklist coverage is complete.

## Change Log
- v2.0 (2026-04-10): Reorganized into documentation policy template and migrated to `doc/architecture`.

## Approvals
- [ ] PM review
- [ ] Architecture team review
