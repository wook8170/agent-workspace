# WJA-21 Kafka 토픽 정의서

## 1. 공통 원칙
- 직렬화 포맷: JSON Schema 기반 JSON (`schemaVersion` 필수)
- 타임존: UTC ISO-8601 (`eventAt`)
- 키 설계:
  - 주문계: `orderId`
  - 입출고계: `receiptNo`/`shipmentNo`
  - 스캔계: `scanId`
- 보존 정책:
  - 이벤트 토픽: 7일
  - 감사/정산 토픽: 90일
- 재처리 정책: 컨슈머 그룹 오프셋 리셋 기반 재처리 + DLQ 분리

## 2. 토픽 목록

| Topic | Producer | Consumer | Key | 파티션 전략 | Retention |
|---|---|---|---|---|---|
| `oms.order.v1` | OMS | WMS, Audit | `orderId` | 주문 단위 순서 보장 목적 hash(orderId) | 7d |
| `oms.order.cancel.v1` | OMS | WMS | `orderId` | hash(orderId) | 7d |
| `wms.receipt.confirmed.v1` | WMS | ERP Adapter, OMS | `receiptNo` | 센터 분산 목적 hash(centerCode + receiptDate) | 30d |
| `wms.closing.data.v1` | WMS | ERP Adapter, Finance Lake | `centerCode:closingDate` | 월마감 부하 분산 hash(centerCode) | 90d |
| `wms.shipment.invoice.request.v1` | WMS | ERP Adapter | `shipmentNo` | hash(shipmentNo) | 7d |
| `erp.invoice.result.v1` | ERP Adapter | WMS, OMS | `shipmentNo` | hash(shipmentNo) | 30d |
| `barcode.scan.inbound.v1` | Barcode Gateway | WMS | `scanId` | hash(centerCode + deviceId) | 3d |
| `barcode.scan.outbound.v1` | Barcode Gateway | WMS | `scanId` | hash(centerCode + deviceId) | 3d |
| `barcode.sync.delta.v1` | WMS Sync Service | Barcode Gateway | `deviceId` | hash(deviceId) | 1d |
| `*.dlq` | 각 서비스 | 운영자 수동처리 | 원본키 유지 | 원본 토픽과 동일 | 14d |

## 3. 메시지 포맷

### 3.1 `oms.order.v1`
```json
{
  "schemaVersion": "1.0.0",
  "eventId": "evt-20260410-0001",
  "eventAt": "2026-04-10T02:31:20Z",
  "orderId": "OMS-20260410-1001",
  "eventType": "ORDER_CREATED",
  "channel": "B2B_PORTAL",
  "customerCode": "CUST-001",
  "items": [
    {"lineNo": 1, "skuCode": "SKU-APPLE-01", "qty": 10, "uom": "EA"}
  ],
  "delivery": {"zipCode": "06236", "address1": "Seoul ..."}
}
```

### 3.2 `wms.receipt.confirmed.v1`
```json
{
  "schemaVersion": "1.0.0",
  "eventId": "evt-20260410-0892",
  "eventAt": "2026-04-10T07:11:55Z",
  "receiptNo": "RCPT-20260410-0034",
  "centerCode": "FC-ICN-01",
  "erpRefNo": "",
  "items": [
    {"skuCode": "SKU-APPLE-01", "receivedQty": 100, "defectQty": 2}
  ]
}
```

### 3.3 `barcode.scan.outbound.v1`
```json
{
  "schemaVersion": "1.0.0",
  "eventId": "evt-20260410-1201",
  "eventAt": "2026-04-10T09:01:40Z",
  "scanId": "SCN-9f12c3",
  "scanType": "OUTBOUND",
  "centerCode": "FC-BUS-02",
  "deviceId": "MOB-023",
  "operatorId": "USR-4482",
  "barcode": "8809001234567",
  "shipmentNo": "SHP-20260410-2231",
  "qty": 1,
  "locationCode": "P-01-02-03"
}
```

## 4. 에러 처리 및 재시도
- 프로듀서 전송 실패: 지수 백오프(1s, 2s, 4s, 8s, 최대 5회)
- 컨슈머 처리 실패:
  - 비재시도 오류(스키마 오류/필수값 누락): 즉시 DLQ
  - 재시도 가능 오류(DB lock/외부 API timeout): 3회 재시도 후 DLQ
- DLQ 운영:
  - `errorCode`, `errorMessage`, `failedAt`, `sourceTopic`, `payload` 저장
  - 10분 주기 재처리 잡 실행
  - 3회 재처리 실패 시 수동조치 큐로 전환

## 5. 권장 파티션 수(초기)
- `oms.order.v1`: 24
- `wms.receipt.confirmed.v1`: 12
- `wms.closing.data.v1`: 6
- `barcode.scan.*`: 12
- 산정 근거: 피크 5만 order-line/h, 스캔 피크 2천 TPS, 컨슈머 Lag 30초 이하 목표
