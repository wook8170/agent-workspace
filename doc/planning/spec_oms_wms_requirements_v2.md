# OMS WMS Detailed Requirements Specification

## 작성자
- 작성자: 기획자 Agent
- 작성일: 2026-04-13
- 버전: v2.0

## 목적 (Purpose)
OMS, WMS, Master Data Hub의 상세 요구사항을 구현 가능한 수준으로 정의한다.

## 대상 (Audience)
기획, 백엔드, 프론트엔드, QA, 운영, PM

## 목차 (Table of Contents)
- 1. 범위 및 원칙
- 2. 기능 요구사항
- 3. 비기능 요구사항
- 4. 연계 문서
- 5. 운영 절차
- 6. 검증 기준

## 주요 내용

### 1. 범위 및 원칙
- 범위: OMS, WMS, Master Data Hub 및 내부 연계 API
- 비범위: 외부 공장 시스템 실구현(확장 포인트만 정의)
- 원칙:
  - OMS는 주문 의사결정, WMS는 물류 실행, MDH는 기준정보 단일 진실원(SSOT)
  - 동기 API와 비동기 이벤트를 혼합
  - 마스터 데이터는 MDH에서 단방향 배포

### 2. 기능 요구사항

#### 2.1 OMS
- 주문 수집:
  - 채널: B2B_PORTAL, MARKETPLACE, EDI, MANUAL
  - 입력: JSON, CSV, EDI 변환 JSON
  - 필수 필드: `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items[]`
  - 유효성: `qty > 0`, ISO-8601, 고객/품목 MDH 존재, `channel + externalOrderNo` 멱등
- 라우팅:
  - 입력: 배송권역, 가용재고, 컷오프, SLA 등급
  - 우선순위: SLA 충족 > 재고충족률 > 운영비용 > 고정 우선순위
  - 산출: 단일/분할 출고 계획 및 라우팅 사유 코드
- 우선순위 산정:
  - 공식: `priorityScore = (slaWeight * slaUrgency) + (vipWeight * customerTier) + (delayWeight * agingHours) + (penaltyWeight * stockRisk)`
  - 기본 가중치: `0.4/0.2/0.3/0.1`
  - 등급: `CRITICAL(>=80)`, `HIGH(60~79)`, `NORMAL(40~59)`, `LOW(<40)`

#### 2.2 WMS
- 입고 상태: `ASN_RECEIVED -> ARRIVED -> INSPECTED -> PUTAWAY_DONE -> RECEIPT_CONFIRMED`
- 출고 상태: `ALLOCATED -> PICKING -> PACKED -> SHIPPED -> HANDED_OVER`
- 예외: 과입고/부족입고/불량, 피킹 불일치, 파손 이벤트 처리
- 재고 조회:
  - 기준: `onHand`, `available`, `allocated`, `inTransit`
  - SLA: 95p 300ms
  - 캐시: TTL 3초, 이벤트 기반 즉시 무효화, 불일치 시 WMS 원장 우선

#### 2.3 Master Data Hub
- 관리 항목:
  - Product: `skuCode`, `skuName`, `category`, `uom`, `temperatureType`, `status`
  - Customer: `customerCode`, `customerName`, `tier`, `billingCycle`, `status`
  - FulfillmentCenter: `centerCode`, `region`, `cutoffTime`, `capacityLimit`, `status`
  - RoutingRule: `ruleId`, `priority`, `conditionExpr`, `targetCenter`, `effectiveFrom`, `effectiveTo`
- 유효성:
  - 코드: 영문 대문자/숫자/`-`
  - 기간: `effectiveFrom <= effectiveTo`
  - 상태: `ACTIVE`, `INACTIVE`, `DEPRECATED`
- 동기화:
  - MDH -> OMS/WMS 단방향
  - 실시간 이벤트 + 1일 1회 전체 스냅샷
  - 스키마 불일치 시 수신 거부 및 경보

### 3. 비기능 요구사항
- 성능:
  - 주문 수집 API 95p 500ms
  - 재고 조회 API 95p 300ms
  - 시간당 주문 처리량 최소 50,000 라인
- 가용성: 월 99.9%, RTO 30분, RPO 5분, Active-Standby 전환
- 보안: OAuth2 Client Credentials + JWT, RBAC, TLS 1.2+, AES-256, 감사로그 1년
- 확장성: 센터/공장 추가 온보딩, 라우팅 룰 무중단 반영, API 서버 수평 확장

### 4. 연계 문서
- 프로세스 흐름: `doc/planning/spec_oms_wms_process_flow_v1.md`
- 데이터 모델: `doc/planning/spec_oms_wms_data_model_v1.md`
- API 계약: `doc/planning/spec_oms_wms_api_contract_v2.md`

### 5. 운영 절차
- 시작: MDH 스냅샷 확인 -> OMS/WMS 헬스체크 -> 큐 적체 점검
- 종료: 신규 수집 차단 -> 진행 주문 drain -> 큐 flush
- 장애(Sev1): 5분 내 온콜, 15분 내 우회/롤백, 30분 내 공지
- 모니터링 임계값:
  - 라우팅 실패율 1% 초과 경보
  - 주문 큐 지연 5분 초과 경보
- 백업: 증분 5분, 전체 1일 1회, 월 1회 DR 리허설

### 6. 검증 기준
- 문서만으로 API/데이터/프로세스 구현 가능
- 시스템 경계(OMS/WMS/MDH) 충돌 없음
- PM/기획/QA 체크리스트 충족

## 변경 이력 (Change Log)
- v2.0 (2026-04-13): 문서 정책 포맷으로 재정비 및 기존 WJA-18 요구사항 반영

## 승인 현황 (Approvals)
- [ ] PM 검토
- [ ] 기획/개발/QA 검토
