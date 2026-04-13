# OMS WMS API Contract Specification

## 작성자
- 작성자: 기획자 Agent
- 작성일: 2026-04-13
- 버전: v2.0

## 목적 (Purpose)
OMS/WMS/MDH 인터페이스 계약을 표준화해 개발과 QA 검증 기준을 통일한다.

## 대상 (Audience)
백엔드, 프론트엔드, QA, 운영, PM

## 목차 (Table of Contents)
- 1. API 개요
- 2. 엔드포인트 계약
- 3. 공통 스키마
- 4. 보안 정책
- 5. 오류 처리

## 주요 내용

### 1. API 개요
- OpenAPI 버전: `3.0.3`
- 도메인: OMS, WMS, Master Data
- 기본 서버: `https://api.example.com`
- 인증: Bearer JWT

### 2. 엔드포인트 계약

#### 2.1 OMS
- `POST /v1/oms/orders` 주문 수집
  - 요청 필수: `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items[]`
  - 성공 응답: `201`, `orderId`, `status=RECEIVED`
  - 오류 응답: `400` Validation Failed
- `POST /v1/oms/orders/{orderId}/route` 라우팅 실행
  - 성공 응답: `200`, `selectedCenters[]`, `reasonCode`
- `POST /v1/oms/orders/{orderId}/priority` 우선순위 계산
  - 성공 응답: `200`, `priorityScore`, `priorityClass`

#### 2.2 WMS
- `POST /v1/wms/receipts` 입고 진행 이벤트 반영
- `POST /v1/wms/shipments` 출고 진행 이벤트 반영
- `GET /v1/wms/inventory/{skuCode}` 실시간 재고 조회

#### 2.3 Master Data
- `GET /v1/master-data/products` 상품 마스터 조회

### 3. 공통 스키마
- `OrderCollectRequest`
  - 필수: `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items`
  - 채널 enum: `B2B_PORTAL`, `MARKETPLACE`, `EDI`, `MANUAL`
- `PriorityResponse`
  - `priorityClass`: `CRITICAL`, `HIGH`, `NORMAL`, `LOW`
- `ReceiptRequest.eventType`
  - `ARRIVED`, `INSPECTED`, `PUTAWAY_DONE`, `RECEIPT_CONFIRMED`

### 4. 보안 정책
- `Authorization: Bearer <JWT>` 헤더 필수
- OAuth2 Client Credentials 기반 토큰 발급
- 역할 기반 인가(RBAC) 적용

### 5. 오류 처리
- 입력 오류: `400`
- 인증 실패: `401`
- 권한 없음: `403`
- 리소스 없음: `404`
- 서버 오류: `500`
- 응답 바디는 공통 `ErrorResponse` 스키마 사용

## 변경 이력 (Change Log)
- v2.0 (2026-04-13): 문서 정책 포맷으로 재정비 및 WJA-18 OpenAPI 계약 반영

## 승인 현황 (Approvals)
- [ ] PM 검토
- [ ] 백엔드/QA 검토
