# Architecture API Contract v1

## Author
- Author: Analysis/Design Agent
- Date: 2026-04-10
- Version: v1.0

## Purpose
Provide API contract index and governance context for the architecture-level OpenAPI specification.

## Audience
Backend developers, frontend developers, integration engineers, and QA engineers.

## Table of Contents
- 1. Specification Artifact
- 2. API Scope Summary
- 3. Validation and Governance

## Main Content

### 1. Specification Artifact
- OpenAPI file: `arch_api_contract_v1.yaml`
- Format: OpenAPI 3.x YAML
- Coverage:
  - OMS order intake and query APIs
  - OMS-WMS allocation and status synchronization
  - MDH lookup and validation APIs

### 2. API Scope Summary
- Authentication and authorization:
  - OAuth2 client credential flow
  - JWT-based API access
- Interface principles:
  - Idempotency for order intake API
  - Standardized error payload (field-level errors)
  - Versioned API base path and backward compatibility policy
- Operational targets:
  - OMS intake API 95p <= 500ms
  - Inventory query API 95p <= 300ms

### 3. Validation and Governance
- Contract-first implementation is required.
- Any breaking change must increment major API version and update change log.
- CI gate should validate schema syntax and required fields before merge.

## Change Log
- v1.0 (2026-04-10): Added policy-compliant API contract index and linked OpenAPI artifact.

## Approvals
- [ ] PM review
- [ ] Architecture team review
