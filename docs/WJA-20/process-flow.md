# WJA-20 시스템별 프로세스 흐름도

## 1. OMS 주문 수집/할당 상세 프로세스

### 1.1 표준 플로우
```mermaid
flowchart LR
    A[1. 채널 주문 수신] --> B[2. 포맷 표준화]
    B --> C[3. 마스터 검증]
    C --> D{4. 유효성 통과?}
    D -- No --> E[5. 거절/에러코드 반환]
    D -- Yes --> F[6. 멱등키 중복검사]
    F --> G{7. 신규 주문?}
    G -- No --> H[8. 기존 주문 조회 반환]
    G -- Yes --> I[9. 우선순위 계산]
    I --> J[10. 라우팅 엔진]
    J --> K[11. WMS 할당요청]
    K --> L[12. 주문상태 ALLOCATED]
    L --> M[13. 이벤트 발행]
```

### 1.2 예외 플로우
- 재고 부족: 대체센터 재탐색 -> 실패 시 `BACKORDER_REQUESTED`
- 라우팅 불가: 폴백 룰 적용 -> 실패 시 수동검토 큐
- 채널 중복 전송: 멱등 처리 후 기존 결과 재응답

## 2. WMS 입출고/재고 상세 프로세스

### 2.1 입고 프로세스
```mermaid
stateDiagram-v2
    [*] --> ASN_RECEIVED
    ASN_RECEIVED --> ARRIVED: 도착등록
    ARRIVED --> INSPECTED: 수량/품질검수
    INSPECTED --> PUTAWAY_DONE: 로케이션 적치
    PUTAWAY_DONE --> RECEIPT_CONFIRMED: 입고확정
    INSPECTED --> EXCEPTION: 과부족/불량
    EXCEPTION --> RECEIPT_CONFIRMED: 조정 후 확정
```

### 2.2 출고 프로세스
```mermaid
stateDiagram-v2
    [*] --> ALLOCATED
    ALLOCATED --> PICKING: 피킹지시
    PICKING --> PACKED: 패킹완료
    PACKED --> SHIPPED: 출고검수
    SHIPPED --> HANDED_OVER: 운송사 인계
    PICKING --> EXCEPTION: 피킹불일치
    PACKED --> EXCEPTION: 포장파손
    EXCEPTION --> PICKING: 재작업
```

### 2.3 재고 데이터 흐름
```mermaid
flowchart LR
    Scan[PDA 바코드 스캔] --> Move[Movement 생성]
    Move --> Ledger[재고원장 반영]
    Ledger --> Cache[Redis 캐시 갱신]
    Ledger --> Event[InventoryChanged 이벤트]
    Event --> OMSRead[OMS 가용재고 조회]
    Event --> ERPFeed[ERP 재고 인터페이스]
```

## 3. ERP 연계 프로세스

### 3.1 송장/입고/마감 연계 시퀀스
```mermaid
sequenceDiagram
    participant OMS as OMS
    participant WMS as WMS
    participant ADP as ERP Adapter
    participant ERP as 더존 ERP
    participant DLQ as Retry/DLQ

    OMS->>ADP: 주문확정/마감 데이터(Outbox)
    WMS->>ADP: 입출고/송장 데이터(Outbox)
    ADP->>ERP: REST/SFTP 전송
    ERP-->>ADP: ACK/결과코드
    ADP-->>OMS: 주문마감 반영 결과
    ADP-->>WMS: 재고/입고 반영 결과

    alt 전송 실패
      ADP->>ADP: 재시도(지수백오프)
      ADP->>DLQ: 최대 재시도 초과 적재
    end
```

### 3.2 동기화 정책
- 실시간: 중요 이벤트(출고확정, 재고조정, 마감확정) 즉시 전송
- 배치: 일 마감 파일 `D+1 00:10` 생성 후 전송
- 정합성: ERP ACK 기준으로 상태 확정, 미수신건은 보류 큐

## 4. Master Data Hub 프로세스

### 4.1 마스터 승인/배포
```mermaid
flowchart LR
    Draft[1. 초안 작성] --> Review[2. 검토]
    Review --> Approve{3. 승인?}
    Approve -- No --> Reject[반려/수정요청]
    Approve -- Yes --> Version[4. 버전 발행]
    Version --> Event[5. 변경 이벤트 발행]
    Version --> Snapshot[6. 일배치 스냅샷]
    Event --> OMS[7. OMS 반영]
    Event --> WMS[8. WMS 반영]
    Snapshot --> OMS
    Snapshot --> WMS
```

### 4.2 데이터 품질 기준
- 코드 규칙: `^[A-Z0-9-]+$`
- 유효기간: `effective_from <= effective_to`
- 참조무결성: 고객/상품/센터 참조 키 사전 검증
- 배포 차단: 품질 규칙 실패 시 승인 불가

## 5. 운영 체크포인트
- OMS 수집 실패율 1% 초과 시 채널별 제한 모드
- WMS 재고정합률 99.5% 미만 시 실사 태스크 자동 생성
- ERP 연계 누락건 10분 이상 적체 시 온콜 알림
- MDH 배포 실패 시 이전 안정버전 자동 롤백
