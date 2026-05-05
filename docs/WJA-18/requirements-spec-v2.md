# WJA-18 상세 요구사항 명세서 v2

## 1. 문서 정보
- 문서명: 상세 요구사항 명세서 v2 (Phase 1)
- 기준 이슈: WJA-18
- 연계 아키텍처: WJA-10 (OMS, WMS, Master Data Hub 분리 구조)
- 목적: 개발팀이 본 문서만으로 구현 가능한 수준의 기능/인터페이스/데이터/운영 요구사항 제공

## 2. 범위 및 원칙
- 범위: OMS, WMS, Master Data Hub 및 내부 연계 API
- 비범위: 외부 공장 연계 실구현(단, 확장 포인트는 본 문서에 정의)
- 설계 원칙:
  - 서비스 경계 명확화: OMS(주문 의사결정), WMS(물류 실행), MDH(기준정보)
  - 동기 API + 비동기 이벤트 혼합
  - 마스터 데이터 단일 진실원(SSOT)

## 3. 기능 요구사항

### 3.1 OMS 상세 기능

#### 3.1.1 주문 수집
- 지원 채널: `B2B_PORTAL`, `MARKETPLACE`, `EDI`, `MANUAL`
- 입력 포맷:
  - JSON (REST)
  - CSV (배치 업로드)
  - EDI X12 (변환 어댑터 통해 내부 JSON 변환)
- 필수 입력 필드:
  - `channel`, `externalOrderNo`, `orderDateTime`, `customerCode`, `items[]`
  - `items[].skuCode`, `items[].qty`, `items[].uom`
- 유효성 규칙:
  - `externalOrderNo` 채널 단위 유일성
  - `qty > 0`, 정수
  - `orderDateTime` ISO-8601
  - 고객/품목은 MDH에 존재해야 함
  - 중복 수신 시 멱등 처리(`channel + externalOrderNo` 키)
- 처리 결과:
  - 성공: OMS 주문번호 발번 및 `RECEIVED`
  - 실패: 에러 코드와 필드 단위 원인 반환

#### 3.1.2 라우팅
- 라우팅 규칙 엔진 입력:
  - 주문 배송지 권역
  - 센터 가용 재고
  - 센터 컷오프 시간
  - 배송 SLA 등급
- 센터 결정 로직:
  - 1순위: SLA 충족 가능한 센터
  - 2순위: 재고 충족률 높은 센터
  - 3순위: 운영비용 낮은 센터
  - 동률: 고정 우선순위 테이블
- 산출:
  - 단일 센터 할당 또는 분할 출고 계획
  - 라우팅 사유 코드 기록

#### 3.1.3 우선순위 산정
- 점수식:
  - `priorityScore = (slaWeight * slaUrgency) + (vipWeight * customerTier) + (delayWeight * agingHours) + (penaltyWeight * stockRisk)`
- 기본 가중치:
  - `slaWeight=0.4`, `vipWeight=0.2`, `delayWeight=0.3`, `penaltyWeight=0.1`
- 점수 구간:
  - `>= 80`: `CRITICAL`
  - `60~79`: `HIGH`
  - `40~59`: `NORMAL`
  - `< 40`: `LOW`
- 가중치는 운영자 화면에서 버전 관리 가능해야 함

### 3.2 WMS 상세 기능

#### 3.2.1 입고
- 절차:
  - ASN 수신
  - 도착 등록
  - 검수(수량/품질)
  - 로케이션 배치
  - 입고 확정
- 상태 전이:
  - `ASN_RECEIVED -> ARRIVED -> INSPECTED -> PUTAWAY_DONE -> RECEIPT_CONFIRMED`
- 예외:
  - 과입고/부족입고/불량 발생 시 예외 코드 생성 및 OMS 통지

#### 3.2.2 출고
- 절차:
  - OMS 할당 수신
  - 피킹 지시
  - 패킹
  - 출고 검수
  - 운송사 인계
- 상태 전이:
  - `ALLOCATED -> PICKING -> PACKED -> SHIPPED -> HANDED_OVER`
- 규칙:
  - 동일 주문은 센터 내 단일 파동(wave) 우선
  - 긴급도 높은 주문은 별도 즉시 피킹 큐 할당

#### 3.2.3 재고 조회/캐시
- 재고 조회 기준:
  - `onHand`, `available`, `allocated`, `inTransit`
- 실시간 조회 SLA:
  - API 응답 95p 300ms 이내
- 캐시 전략:
  - 읽기 캐시 TTL 3초
  - 재고 변동 이벤트 수신 시 즉시 무효화
  - 캐시 불일치 시 WMS 원장 값 우선

### 3.3 Master Data Hub 상세 기능

#### 3.3.1 마스터 항목 및 필드
- 제품(`Product`): `skuCode`, `skuName`, `category`, `uom`, `temperatureType`, `status`
- 고객(`Customer`): `customerCode`, `customerName`, `tier`, `billingCycle`, `status`
- 센터(`FulfillmentCenter`): `centerCode`, `region`, `cutoffTime`, `capacityLimit`, `status`
- 라우팅규칙(`RoutingRule`): `ruleId`, `priority`, `conditionExpr`, `targetCenter`, `effectiveFrom`, `effectiveTo`

#### 3.3.2 데이터 유효성
- 코드성 키는 영문 대문자/숫자/`-`만 허용
- `effectiveFrom <= effectiveTo`
- 상태값은 enum 관리 (`ACTIVE`, `INACTIVE`, `DEPRECATED`)

#### 3.3.3 동기화 정책
- MDH -> OMS/WMS 단방향 배포
- 배포 방식:
  - 실시간 변경 이벤트
  - 1일 1회 전체 스냅샷
- 충돌 처리:
  - 최신 버전 우선
  - 스키마 불일치 시 수신 거부 및 경보

## 4. 비기능 요구사항

### 4.1 성능
- 주문 수집 API: 95p 500ms 이하
- 재고 조회 API: 95p 300ms 이하
- 시간당 주문 처리량: 최소 50,000 라인
- 동시 사용자: 백오피스 300명, API 클라이언트 2,000 커넥션

### 4.2 가용성
- 월 가용성 SLA: 99.9%
- RTO: 30분 이내
- RPO: 5분 이내
- 장애 시 Active-Standby 전환 지원

### 4.3 보안
- 인증: OAuth2 Client Credentials + JWT
- 인가: 역할 기반 접근 제어(RBAC)
- 감사: 주요 변경/승인/삭제 이벤트 1년 보관
- 암호화:
  - 전송구간 TLS 1.2+
  - 저장구간 AES-256

### 4.4 확장성
- 외부 공장/센터 추가 시 마스터 등록만으로 온보딩 가능
- 라우팅 엔진 룰 추가 시 무중단 반영
- OMS/WMS는 수평 확장 가능한 무상태 API 서버 구조

## 5. 인터페이스/데이터/프로세스 연결
- REST API 상세: `openapi.yaml` 참조
- 데이터 모델/ERD: `data-model.md` 참조
- 주문-배송 상세 절차 및 예외 흐름: `process-flow.md` 참조

## 6. 운영 절차

### 6.1 시작/종료
- 시작:
  - MDH 스냅샷 적재 확인
  - OMS/WMS 헬스체크 정상 확인
  - 큐 적체 0 확인 후 수집 오픈
- 종료:
  - 신규 수집 차단
  - 처리 중 주문 drain
  - 큐 flush 후 종료

### 6.2 장애 대응
- 심각도 정의: Sev1~Sev3
- Sev1 대응:
  - 5분 내 온콜 호출
  - 15분 내 임시 우회 또는 롤백
  - 30분 내 고객 공지

### 6.3 모니터링/임계값
- 지표:
  - API 95p/99p 지연
  - 주문 큐 적체 건수
  - 재고 불일치율
  - 라우팅 실패율
- 기본 임계값:
  - 라우팅 실패율 1% 초과 시 경보
  - 주문 큐 지연 5분 초과 시 경보

### 6.4 백업/복구
- DB 증분 백업: 5분
- 전체 백업: 1일 1회
- 월 1회 DR 리허설 수행 및 결과 기록

## 7. 검증 기준(완료 정의)
- 개발팀이 문서 기반으로 API/데이터/프로세스 구현 가능
- WJA-10 아키텍처 경계와 충돌 없음
- PM/분석설계/고객 검토 체크리스트 100% 충족
