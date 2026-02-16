# 이벤트 카탈로그 (Event Catalog)

**목적**: 60개 지점 무인 운영 시스템의 모든 이벤트 정의 및 처리 규칙
**작성일**: 2026-02-16

---

## 이벤트 카탈로그 표

| # | EventName | Producer | Consumer | PayloadKey | Retention | SLA/Severity |
|---|-----------|----------|----------|-----------|-----------|--------------|
| **출입 이벤트** |
| 1 | `access.entry.requested` | BLE 도어락 | Edge Gateway | userId, doorId, timestamp, method(BLE/QR/Card) | 90일 | P1 / 100ms |
| 2 | `access.entry.granted` | Edge Gateway | Cloud Access, Monitoring | userId, doorId, timestamp, authMode(online/offline) | 90일 | P1 / 500ms |
| 3 | `access.entry.denied` | Edge Gateway | Cloud Access, Monitoring | userId, doorId, timestamp, reason(noPermission/expired/blacklist) | 90일 | P0 / 200ms |
| 4 | `access.exit.detected` | PIR 센서 | Edge Gateway | doorId, timestamp, direction(exit) | 30일 | P2 / 1s |
| 5 | `access.tailgating.detected` | IP 카메라 | Edge Gateway, Monitoring | doorId, timestamp, suspectedCount | 90일 | P0 / 3s |
| 6 | `access.forcedEntry.detected` | 도어 센서 | Edge Gateway, Monitoring | doorId, timestamp, alarmLevel(critical) | 365일 | P0 / 즉시 |
| **예약 이벤트** |
| 7 | `booking.created` | 모바일 앱 | Cloud Booking, Edge Gateway | bookingId, userId, resourceId(room/booth), startTime, duration | 365일 | P2 / 2s |
| 8 | `booking.checkedIn` | 모바일 앱 | Cloud Booking, Edge Gateway, Billing | bookingId, userId, actualStartTime | 365일 | P1 / 1s |
| 9 | `booking.checkedOut` | 모바일 앱 / 센서 | Cloud Booking, Billing | bookingId, userId, actualEndTime, actualDuration | 365일 | P1 / 1s |
| 10 | `booking.noShow` | Edge Gateway | Cloud Booking, Monitoring | bookingId, userId, scheduledTime | 365일 | P2 / 5s |
| 11 | `booking.cancelled` | 모바일 앱 | Cloud Booking, Edge Gateway | bookingId, userId, cancelTime, reason | 365일 | P2 / 2s |
| 12 | `booking.extended` | 모바일 앱 | Cloud Booking, Billing | bookingId, userId, newEndTime, additionalFee | 365일 | P1 / 1s |
| **결제 이벤트** |
| 13 | `billing.charge.initiated` | Cloud Billing | PG 게이트웨이 | chargeId, userId, amount, currency(KRW), items[] | 7년 | P1 / 3s |
| 14 | `billing.charge.completed` | PG 게이트웨이 | Cloud Billing, 모바일 앱 | chargeId, transactionId, status(success), timestamp | 7년 | P1 / 5s |
| 15 | `billing.charge.failed` | PG 게이트웨이 | Cloud Billing, Monitoring | chargeId, userId, reason(insufficient/cardError), retryCount | 7년 | P0 / 3s |
| 16 | `billing.refund.requested` | 관제 콘솔 | Cloud Billing, PG 게이트웨이 | refundId, originalChargeId, amount, reason | 7년 | P1 / 5s |
| **설비 이벤트** |
| 17 | `facility.occupancy.changed` | PIR 센서 / 점유센서 | Edge Gateway, Cloud Booking | resourceId, occupancy(true/false), timestamp | 30일 | P2 / 5s |
| 18 | `facility.environment.alert` | 환경 센서 | Edge Gateway, Monitoring | sensorId, type(temp/humidity/co2), value, threshold | 90일 | P1 / 10s |
| 19 | `facility.light.controlChanged` | 조명 컨트롤러 | Edge Gateway, Data Module | controllerId, zoneId, state(on/off/dim), brightness(0-100) | 7일 | P3 / 30s |
| 20 | `facility.hvac.statusChanged` | HVAC 컨트롤러 | Edge Gateway, Data Module | hvacId, mode(cool/heat/auto), setpoint, currentTemp | 30일 | P2 / 30s |
| 21 | `facility.power.anomaly` | 전력 모니터 | Edge Gateway, Monitoring | meterId, phase, voltage, current, anomalyType(surge/drop) | 90일 | P0 / 10s |
| **보안 이벤트** |
| 22 | `security.camera.motionDetected` | IP 카메라 | Edge Gateway, Monitoring | cameraId, zoneId, timestamp, confidence(0-100) | 30일 | P2 / 5s |
| 23 | `security.alarm.triggered` | 통합 보안 시스템 | Edge Gateway, Monitoring, 외부 보안업체 | alarmId, type(intrusion/fire/emergency), location, severity(critical) | 365일 | P0 / 즉시 |
| 24 | `security.lockdown.initiated` | 관제 콘솔 | Edge Gateway, All Doors | siteId, reason(security/disaster), timestamp | 365일 | P0 / 즉시 |
| 25 | `security.credential.suspicious` | Cloud Access | Monitoring | userId, reason(multipleFailures/geoAnomaly), attemptCount | 365일 | P0 / 3s |
| **에너지 이벤트** |
| 26 | `energy.consumption.reported` | 전력 모니터 | Cloud Data Module | meterId, interval(15min), kwh, cost(KRW) | 7년 | P3 / 1min |
| 27 | `energy.optimization.triggered` | Cloud Policy | Edge Gateway, 조명/HVAC | siteId, action(dimLight/adjustHVAC), reason(lowOccupancy), savings(%) | 30일 | P3 / 30s |
| **장애 이벤트** |
| 28 | `system.edge.offline` | Cloud Monitoring | 관제 콘솔, SMS 알림 | siteId, lastHeartbeat, offlineDuration(sec) | 365일 | P0 / 즉시 |
| 29 | `system.edge.online` | Edge Gateway | Cloud Monitoring | siteId, timestamp, queuedEventCount | 365일 | P1 / 3s |
| 30 | `system.device.unreachable` | Edge Gateway | Cloud Monitoring | deviceId, type(door/camera/sensor), lastSeen, siteId | 90일 | P1 / 30s |
| 31 | `system.device.batteryLow` | 무선 센서 | Edge Gateway, Monitoring | deviceId, batteryLevel(%), estimatedDaysLeft | 90일 | P2 / 1h |
| 32 | `system.sync.failed` | Edge Gateway | Cloud Monitoring | siteId, failedEventCount, errorType(network/auth/quota) | 90일 | P1 / 1min |

---

## 이벤트 처리 정책

### Retention (보관 기간)

- **7일**: 실시간 운영 메트릭 (조명/온도 등)
- **30일**: 일반 센서 데이터 (점유, 모션)
- **90일**: 출입/보안 로그 (개인정보보호법 준수)
- **365일**: 예약/보안 사고 (감사 추적)
- **7년**: 결제 데이터 (전자상거래법, 세법)

### SLA (Service Level Agreement)

- **P0 (Critical)**: 즉시 ~ 3초 - 보안/재난/결제 실패
- **P1 (High)**: 100ms ~ 1초 - 출입/예약 핵심 흐름
- **P2 (Medium)**: 2초 ~ 30초 - 일반 이벤트
- **P3 (Low)**: 30초 ~ 1시간 - 배치성 데이터

### Severity (심각도)

- **Critical**: 즉시 알림 (SMS/전화) + 관제 대응 필수
- **High**: 관제 콘솔 알림 + 15분 내 확인
- **Medium**: 대시보드 표시, 일일 리포트 포함
- **Low**: 로그 기록, 주간 리포트 포함

---

## 이벤트 페이로드 공통 필드

모든 이벤트는 다음 공통 필드 포함:

```json
{
  "eventId": "uuid-v4",
  "eventName": "access.entry.granted",
  "version": "1.0",
  "timestamp": "2026-02-16T14:30:45.123Z",
  "source": {
    "type": "edge|cloud|device",
    "id": "edge-gangnam-01",
    "siteId": "site-gangnam"
  },
  "payload": {
    // 이벤트별 고유 데이터
  },
  "metadata": {
    "correlationId": "uuid-v4",  // 연관 이벤트 추적
    "causationId": "uuid-v4",    // 원인 이벤트 ID
    "userId": "user-12345"       // 행위자 (옵션)
  }
}
```

---

## 이벤트 라우팅 규칙

### Real-time Stream (< 1초)
- 출입/예약/보안 이벤트
- 대상: Edge ↔ Cloud 동기화, 관제 콘솔

### Batch Processing (분/시간 단위)
- 에너지 소비, 사용 패턴 분석
- 대상: Data Warehouse, BI 대시보드

### Alert Routing (조건부)
- P0 이벤트: SMS + Slack + 전화 (3채널 동시)
- P1 이벤트: 관제 콘솔 알림 + Slack
- P2 이벤트: 관제 콘솔만
- P3 이벤트: 로그 기록만

---

## 오프라인 이벤트 처리

### Edge 로컬 큐잉
- 네트워크 단절 시 모든 이벤트를 로컬 SSD에 저장
- 최대 보관: 48시간 (약 100만 이벤트, 1GB 추정)
- 재연결 시 배치 전송 (100개/batch, 중복 제거)

### Cloud 중복 제거
- `eventId` 기반 멱등성(Idempotency) 보장
- 7일 내 동일 eventId 재전송 시 무시
- 순서 보장: `timestamp` + `correlationId` 기반 정렬

---

**다음 단계**: 이벤트 스키마 상세 정의 (JSON Schema), 에러 핸들링 시나리오, 테스트 케이스 작성
