# 60개 지점 무인 운영 아키텍처 다이어그램

## A. 전체 시스템 컨텍스트 다이어그램

```mermaid
graph TB
    subgraph "Device Layer"
        D1[출입통제<br/>BLE/RFID 리더<br/>전기 도어락]
        D2[IP 카메라<br/>PIR 센서<br/>환경 센서]
        D3[조명/HVAC<br/>전력 모니터]
        D4[키오스크<br/>QR 스캐너]
    end

    subgraph "Edge Layer (지점 x60)"
        E1[Edge Gateway]
        E2[권한 캐시<br/>SQLite/Redis]
        E3[이벤트 버퍼<br/>48h 로컬 저장]
    end

    subgraph "Cloud Layer"
        C1[Access Module<br/>출입 권한 관리]
        C2[Booking Module<br/>예약 엔진]
        C3[Billing Module<br/>정산]
        C4[Monitoring Module<br/>중앙 관제]
        C5[Policy Module<br/>정책 배포]
        C6[Data Module<br/>이벤트 스트림]
    end

    subgraph "Console Layer"
        U1[본사 관제팀<br/>전체 관리]
        U2[지점 매니저<br/>지점별 관리]
        U3[외주 업체<br/>설비 알림]
    end

    subgraph "3rd Party"
        T1[결제 게이트웨이<br/>PG사]
        T2[SMS/Push 알림<br/>Notification Service]
        T3[외부 인증<br/>OAuth/SAML]
    end

    D1 -->|MQTT| E1
    D2 -->|MQTT| E1
    D3 -->|Modbus/MQTT| E1
    D4 -->|HTTP| E1

    E1 <-->|gRPC<br/>권한 동기화| C1
    E2 -->|로컬 검증| E1
    E3 -->|재전송| C6

    C1 -->|권한 정책| C5
    C2 -->|예약 데이터| C6
    C3 -->|정산 요청| T1
    C4 -->|알림 발송| T2
    C6 -->|이벤트 수집| C4

    U1 -->|관리| C4
    U2 -->|관리| C4
    U3 -->|조회| C4

    C1 -.->|인증| T3

    style E1 fill:#e1f5ff
    style C6 fill:#fff4e1
    style C4 fill:#ffe1e1
```

---

## B. 지점(Edge) 내부 구성 다이어그램

```mermaid
graph TB
    subgraph "Site: 강남점"
        subgraph "Zone: 로비"
            Z1D1[정문 도어락]
            Z1D2[로비 카메라 x2]
            Z1D3[키오스크]
        end

        subgraph "Zone: 회의실층"
            Z2D1[회의실 A 도어락]
            Z2D2[회의실 B 도어락]
            Z2D3[복도 PIR 센서 x4]
            Z2D4[회의실 환경센서 x2]
        end

        subgraph "Zone: 프라이빗 오피스"
            Z3D1[오피스 출입문 x5]
            Z3D2[카메라 x3]
        end

        subgraph "Zone: 공용설비"
            Z4D1[폰부스 점유센서 x3]
            Z4D2[리차징존 센서 x2]
            Z4D3[조명 컨트롤러 x10]
            Z4D4[HVAC 컨트롤러]
            Z4D5[전력 모니터]
        end

        subgraph "Edge Infrastructure"
            GW[Edge Gateway<br/>Industrial PC<br/>Docker Runtime]
            SW[PoE Switch<br/>24-port]
            RT[Router<br/>100Mbps + 4G Backup]
            UPS[UPS<br/>4h Backup]
        end
    end

    Z1D1 -->|PoE| SW
    Z1D2 -->|PoE| SW
    Z1D3 -->|PoE| SW

    Z2D1 -->|RS485| GW
    Z2D2 -->|RS485| GW
    Z2D3 -->|Zigbee| GW
    Z2D4 -->|Zigbee| GW

    Z3D1 -->|PoE| SW
    Z3D2 -->|PoE| SW

    Z4D1 -->|Zigbee| GW
    Z4D2 -->|Zigbee| GW
    Z4D3 -->|DALI| GW
    Z4D4 -->|Modbus| GW
    Z4D5 -->|Modbus| GW

    SW -->|Ethernet| GW
    GW -->|WAN| RT
    RT -.->|Internet| Cloud((Cloud))
    RT -.->|4G Backup| Cloud

    UPS -->|전원| GW
    UPS -->|전원| SW
    UPS -->|전원| RT

    style GW fill:#e1f5ff
    style RT fill:#ffe1e1
    style UPS fill:#e1ffe1
```

---

## C. 핵심 이벤트 흐름

### C-1. 출입 이벤트 흐름

```mermaid
sequenceDiagram
    actor User as 회원
    participant App as 모바일 앱
    participant Door as BLE 도어락
    participant Edge as Edge Gateway
    participant Cloud as Cloud Access
    participant Monitor as 관제 콘솔

    User->>App: 지점 도착, 앱 실행
    App->>Door: BLE 신호 브로드캐스트
    Door->>Edge: 출입 요청 (userId, doorId, timestamp)

    alt 온라인 모드
        Edge->>Edge: 로컬 캐시 조회 (4h TTL)
        Edge->>Cloud: 권한 검증 요청 (< 100ms)
        Cloud->>Edge: 권한 승인 (allow/deny)
    else 오프라인 모드
        Edge->>Edge: 로컬 캐시만 사용
        Note over Edge: 캐시 없으면 deny<br/>(Fail-secure)
    end

    Edge->>Door: 도어 오픈 명령
    Door->>User: 도어 언락 (3초)

    Edge->>Cloud: 출입 로그 전송 (비동기)
    Cloud->>Monitor: 실시간 이벤트 표시

    alt 비정상 패턴 감지
        Cloud->>Monitor: 알림 발송 (중복 출입/미등록 시도)
    end
```

### C-2. 회의실 예약 및 체크인 흐름

```mermaid
sequenceDiagram
    actor User as 회원
    participant App as 모바일 앱
    participant Cloud as Booking Module
    participant Edge as Edge Gateway
    participant Sensor as 회의실 센서
    participant Billing as Billing Module

    User->>App: 회의실 예약 요청
    App->>Cloud: POST /booking (roomId, time, duration)
    Cloud->>Cloud: 가용성 검증
    Cloud->>App: 예약 확정 (bookingId)
    Cloud->>Edge: 예약 스케줄 동기화

    Note over User: 예약 시간 도래

    User->>App: 회의실 도착, 체크인
    App->>Edge: 체크인 요청 (bookingId)
    Edge->>Edge: 로컬 스케줄 확인
    Edge->>Sensor: 도어 언락 + 조명 ON

    Sensor->>Edge: 점유 시작 (occupancy=true)
    Edge->>Cloud: 체크인 로그 전송

    alt 노쇼 시나리오
        Note over Sensor: 15분 경과, 점유 없음
        Sensor->>Edge: 점유 없음 (occupancy=false)
        Edge->>Cloud: 노쇼 이벤트
        Cloud->>Cloud: 예약 자동 취소
    end

    Note over User: 사용 종료

    User->>App: 체크아웃
    Sensor->>Edge: 점유 종료 (occupancy=false)
    Edge->>Cloud: 체크아웃 로그 (사용 시간 확정)
    Cloud->>Billing: 과금 요청 (duration, rate)
    Billing->>Cloud: 정산 완료
    Cloud->>App: 영수증 발송
```

### C-3. 장애 및 복구 흐름

```mermaid
sequenceDiagram
    participant Edge as Edge Gateway
    participant Cloud as Cloud
    participant Device as IoT Device
    participant Monitor as 관제

    Note over Edge,Cloud: 정상 운영 중

    Cloud-XEdge: 네트워크 단절 발생
    Edge->>Edge: 오프라인 모드 전환
    Edge->>Device: 로컬 캐시 기반 제어 계속

    Device->>Edge: 이벤트 발생
    Edge->>Edge: 로컬 큐에 저장 (최대 48h)

    Note over Edge: 2시간 경과

    Edge->>Cloud: 재연결 성공
    Edge->>Cloud: 헬스체크 (timestamp, queueSize)
    Cloud->>Edge: 동기화 시작 신호

    loop 이벤트 재전송
        Edge->>Cloud: 큐잉된 이벤트 배치 전송 (100개/batch)
        Cloud->>Edge: ACK (중복 제거 적용)
    end

    Edge->>Cloud: 동기화 완료
    Cloud->>Monitor: 지점 복구 알림 표시

    Cloud->>Edge: 최신 권한 동기화
    Edge->>Edge: 로컬 캐시 갱신

    Note over Edge,Cloud: 정상 운영 재개
```

---

## 다이어그램 렌더링 안내

위 Mermaid 코드는 다음 도구로 렌더링 가능:
- Mermaid Live Editor: https://mermaid.live
- VS Code Mermaid Preview 플러그인
- GitHub/GitLab (자동 렌더링)
- Confluence/Notion Mermaid 블록
