# OMS/WMS Detailed Requirements Specification v2

## 작성자
- 작성자: 기획자 에이전트
- 작성일: 2026-04-10
- 버전: v2.0

## 목적 (Purpose)
OMS, WMS, Master Data Hub 통합 범위의 상세 요구사항을 구현 가능한 수준으로 정리한다.

## 대상 (Audience)
기획, 개발, QA, 운영

## 목차 (Table of Contents)
1. 범위 및 원칙
2. 기능 요구사항
3. 비기능 요구사항
4. 인터페이스/데이터/프로세스 연결
5. 운영 절차
6. 검증 기준

## 주요 내용

### 1. 범위 및 원칙
- 범위: OMS, WMS, Master Data Hub 및 내부 연계 API
- 비범위: 외부 공장 연계 실구현(확장 포인트는 정의)
- 설계 원칙:
  - 서비스 경계: OMS(주문 의사결정), WMS(물류 실행), MDH(기준정보)
  - 동기 API + 비동기 이벤트 혼합
  - 마스터 데이터 SSOT 유지

### 2. 기능 요구사항

#### 2.1 OMS

##### 2.1.1 주문 수집
- 지원 채널: `B2B_PORTAL`, `MARKETPLACE`, `EDI`, `MANUAL`
- 입력 포맷: JSON(REST), CSV(배치), EDI X12(어댑터 변환)
- 필수 입력 필드:
  - `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items[]`
  - `items[].skuCode`, `items[].qty`, `items[].uom`
- 유효성 규칙:
  - `externalOrderNo` 채널 단위 유일
  - `qty > 0` 정수
  - `orderDateTime` ISO-8601
  - 고객/품목 MDH 존재 검증
  - 중복 수신 멱등 처리(`channel + externalOrderNo`)
- 처리 결과:
  - 성공: OMS 주문번호 발번, `RECEIVED`
  - 실패: 에러 코드와 필드 원인 반환

##### 2.1.2 라우팅
- 입력: 배송지 권역, 센터 가용재고, 컷오프, SLA
- 결정 로직:
  - 1순위 SLA 충족
  - 2순위 재고 충족률
  - 3순위 운영비용
  - 동률 시 고정 우선순위
- 산출: 단일/분할 출고 계획, 라우팅 사유 코드

##### 2.1.3 우선순위 산정
- 공식:
  - `priorityScore = (slaWeight * slaUrgency) + (vipWeight * customerTier) + (delayWeight * agingHours) + (penaltyWeight * stockRisk)`
- 기본 가중치: `0.4 / 0.2 / 0.3 / 0.1`
- 점수 구간:
  - `>=80`: `CRITICAL`
  - `60~79`: `HIGH`
  - `40~59`: `NORMAL`
  - `<40`: `LOW`
- 가중치 버전 관리: 운영자 화면에서 변경 및 이력 관리

#### 2.2 WMS

##### 2.2.1 입고
- 절차: ASN 수신 → 도착 등록 → 검수 → 로케이션 배치 → 입고 확정
- 상태: `ASN_RECEIVED -> ARRIVED -> INSPECTED -> PUTAWAY_DONE -> RECEIPT_CONFIRMED`
- 예외: 과/부족입고, 불량 발생 시 예외 코드 생성 + OMS 통지

##### 2.2.2 출고
- 절차: OMS 할당 수신 → 피킹 지시 → 패킹 → 출고 검수 → 운송사 인계
- 상태: `ALLOCATED -> PICKING -> PACKED -> SHIPPED -> HANDED_OVER`
- 규칙:
  - 동일 주문 센터 내 단일 wave 우선
  - 긴급 주문 즉시 피킹 큐 분리

##### 2.2.3 재고 조회/캐시
- 조회 지표: `onHand`, `available`, `allocated`, `inTransit`
- SLA: 95p 300ms 이내
- 캐시:
  - TTL 3초
  - 재고 변동 이벤트 수신 시 즉시 무효화
  - 불일치 시 WMS 원장 우선

#### 2.3 Master Data Hub
- 주요 엔티티: Product, Customer, FulfillmentCenter, RoutingRule
- 유효성:
  - 코드형 키: 영문 대문자/숫자/`-`
  - `effectiveFrom <= effectiveTo`
  - 상태 enum: `ACTIVE`, `INACTIVE`, `DEPRECATED`
- 동기화:
  - MDH → OMS/WMS 단방향
  - 실시간 이벤트 + 1일 1회 전체 스냅샷
  - 스키마 불일치 시 수신 거부 및 경보

### 3. 비기능 요구사항
- 성능:
  - 주문 수집 API 95p 500ms 이하
  - 재고 조회 API 95p 300ms 이하
  - 처리량 시간당 50,000 라인
- 가용성:
  - 월 SLA 99.9%, RTO 30분, RPO 5분
- 보안:
  - OAuth2 Client Credentials + JWT
  - RBAC, 감사로그 1년 보관
  - TLS 1.2+, AES-256
- 확장성:
  - 공장/센터 확장 시 마스터 등록 기반 온보딩
  - 라우팅 룰 무중단 반영

### 4. 인터페이스/데이터/프로세스 연결
- API 계약: `spec_oms_wms_api_contract_v2.md`
- 데이터 모델: `spec_oms_wms_data_model_v1.md`
- 프로세스: `spec_oms_wms_process_flow_v1.md`

### 5. 운영 절차
- 시작: MDH 스냅샷/헬스체크/큐 적체 확인 후 수집 오픈
- 종료: 신규 수집 차단 후 drain/flush
- 장애 대응:
  - Sev1: 5분 내 온콜, 15분 내 우회/롤백, 30분 내 공지
- 모니터링:
  - 라우팅 실패율 1% 초과 경보
  - 주문 큐 지연 5분 초과 경보
- 백업/복구:
  - 증분 5분, 전체 1일 1회, 월 1회 DR 리허설

### 6. 검증 기준(완료 정의)
- 문서 기준으로 API/데이터/프로세스 구현 가능
- 아키텍처 경계와 충돌 없음
- PM/분석설계/고객 검토 체크리스트 충족

## 변경 이력 (Change Log)
- v2.0 (2026-04-10): WJA-23 정책 형식으로 문서 재정비

## 승인 현황 (Approvals)
- [ ] PM 검토
- [ ] 기획/개발/QA 검토
