# OMS/WMS API Contract v2

## 작성자
- 작성자: 기획자 에이전트
- 작성일: 2026-04-10
- 버전: v2.0

## 목적 (Purpose)
OMS/WMS/MDH 연계 API를 정책 준수 형식으로 정리하고 구현/테스트 기준 계약으로 사용한다.

## 대상 (Audience)
백엔드 개발, 프론트엔드(관리자), QA, 운영

## 목차 (Table of Contents)
1. API 개요
2. 엔드포인트 목록
3. 주요 스키마
4. 보안/공통 규칙
5. 예외 응답 기준

## 주요 내용

### 1. API 개요
- OpenAPI 버전: 3.0.3
- 도메인: OMS, WMS, Master Data
- 기본 서버: `https://api.example.com`
- 인증: Bearer JWT (`OAuth2 Client Credentials` 기반)

### 2. 엔드포인트 목록

| Method | Path | Domain | 설명 |
|---|---|---|---|
| POST | `/v1/oms/orders` | OMS | 주문 수집 |
| POST | `/v1/oms/orders/{orderId}/route` | OMS | 주문 라우팅 실행 |
| POST | `/v1/oms/orders/{orderId}/priority` | OMS | 우선순위 계산 |
| POST | `/v1/wms/receipts` | WMS | 입고 단계 진행 |
| POST | `/v1/wms/shipments` | WMS | 출고 단계 진행 |
| GET | `/v1/wms/inventory/{skuCode}` | WMS | 실시간 재고 조회 |
| GET | `/v1/master-data/products` | Master Data | 상품 목록 조회 |

### 3. 주요 스키마

#### 3.1 OrderCollectRequest
- 필수: `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items`
- 채널 enum: `B2B_PORTAL`, `MARKETPLACE`, `EDI`, `MANUAL`
- `items[].qty` 최소값: 1

#### 3.2 PriorityResponse
- `priorityClass` enum: `CRITICAL`, `HIGH`, `NORMAL`, `LOW`

#### 3.3 ShipmentRequest
- 필수: `centerCode`, `shipmentNo`, `eventType`
- `eventType` enum: `PICKING`, `PACKED`, `SHIPPED`, `HANDED_OVER`

### 4. 보안/공통 규칙
- 보안 스키마: `bearerAuth`
- 날짜/시간 필드: ISO-8601
- 오류 응답: `ErrorResponse` 스키마 표준 사용
- 멱등 처리: 주문 수집 시 `channel + externalOrderNo` 기준

### 5. 예외 응답 기준
- `400`: 입력값 유효성 실패
- `401`: 인증 실패
- `403`: 권한 없음
- `404`: 리소스 없음
- `409`: 중복/상태 충돌
- `500`: 서버 내부 오류

## 변경 이력 (Change Log)
- v2.0 (2026-04-10): WJA-23 정책 형식으로 문서 재정비 (기존 `openapi.yaml` 내용 문서화)

## 승인 현황 (Approvals)
- [ ] PM 검토
- [ ] 개발/QA 검토
