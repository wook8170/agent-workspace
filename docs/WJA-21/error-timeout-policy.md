# WJA-21 에러 처리 및 타임아웃 정책

## 1. 목표
- 장애 전파 최소화
- 호출자 재시도 기준 표준화
- 운영자 대응 가능성 확보(에러 코드/추적성)

## 2. 표준 에러 응답
```json
{
  "traceId": "tr-2c1f5e",
  "code": "WMS_50401",
  "message": "ERP timeout",
  "details": [{"field": "receiptNo", "reason": "already processed"}],
  "retryable": true,
  "timestamp": "2026-04-10T10:20:30Z"
}
```

## 3. 에러 코드 체계
- 형식: `{SYSTEM}_{HTTP}{SEQ}`
  - SYSTEM: `OMS`, `WMS`, `ERP`, `BARCODE`, `COMMON`
  - 예: `OMS_40001`, `WMS_40902`, `ERP_50401`

## 4. 주요 에러 매핑

| 상황 | HTTP | 코드 | 재시도 |
|---|---:|---|---|
| 필수값 누락 | 400 | `COMMON_40001` | N |
| 형식 오류(날짜/enum) | 400 | `COMMON_40002` | N |
| 인증 실패 | 401 | `COMMON_40101` | N |
| 권한 부족 | 403 | `COMMON_40301` | N |
| 데이터 없음 | 404 | `COMMON_40401` | N |
| 중복 요청 | 409 | `OMS_40901` | N |
| 버전 충돌 | 409 | `OMS_40902` | 조건부 |
| 업무 규칙 위반 | 422 | `WMS_42201` | N |
| 외부시스템 임시 장애 | 502 | `ERP_50201` | Y |
| 외부시스템 타임아웃 | 504 | `ERP_50401` | Y |
| 내부 미처리 예외 | 500 | `COMMON_50001` | Y(제한적) |

## 5. 타임아웃/재시도 기준

| 인터페이스 | Connect Timeout | Read Timeout | 재시도 횟수 | 백오프 |
|---|---:|---:|---:|---|
| OMS -> WMS (주문) | 1s | 3s | 2 | 200ms, 500ms |
| WMS -> ERP (입고/마감) | 2s | 5s | 3 | 500ms, 1s, 2s |
| Barcode -> WMS (스캔) | 1s | 2s | 2 | 100ms, 300ms |
| ERP 조회 API | 2s | 4s | 1 | 500ms |

## 6. Circuit Breaker 정책
- 대상: WMS->ERP, OMS->WMS 핵심 호출
- 기준:
  - 60초 윈도우 오류율 50% 초과 시 Open
  - Open 상태 30초 유지 후 Half-Open 전환
  - Half-Open 샘플 10건 성공 시 Close

## 7. 비동기 실패 정책(Kafka)
- 재시도 가능 오류: 최대 3회 재처리
- 스키마/데이터 오류: 즉시 DLQ
- DLQ 알림: Slack/PagerDuty 5분 누적 20건 이상
- 수동 복구 SOP:
  1. 원인 분류(데이터/시스템/코드)
  2. 정정 후 재처리 배치 실행
  3. 재발 방지 항목 RCA 등록

## 8. 운영 가이드
- 모든 에러 로그에 `traceId`, `requestId`, `principal` 기록
- PII 필드(`phone`, `address`)는 로그 마스킹
- Sev1 기준:
  - 5분 이상 주문 생성 실패율 10% 초과
  - ERP 연계 타임아웃 연속 10분
