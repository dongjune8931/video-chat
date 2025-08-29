<img width="1512" height="982" alt="image" src="https://github.com/user-attachments/assets/e5688537-cc8d-470d-b195-9eec7a477eec" />


# Spring WebRTC Signaling Server (STOMP)

스프링(백엔드)만으로 구성된 WebRTC 1:1 화상통화용 시그널링 서버입니다. 미디어(오디오·영상)는 브라우저 간 P2P로 전송되고, 서버는 **방 상태 관리**와 **시그널 중계**만 담당합니다.

---

##  핵심 특징

* **STOMP over WebSocket** 기반 시그널링
* 방 인원 관리 (정원 2명), 중복 입장 방지
* 인원이 1명에서 2명이 되는 순간 **`READY`** 메시지를 1회만 브로드캐스트하여 동시 `OFFER` 충돌 완화
* 개인 큐(`/user/queue/*`)로 시스템 메시지 및 내 세션 ID 개별 전달
* 연결 종료 또는 브라우저 닫힘 감지 시 세션 자동 정리

---

##  아키텍처 개요

* **브라우저 (WebRTC)**: 카메라·마이크 확보, 피어 연결(`Offer`/`Answer`), ICE 교환, 미디어 전송
* **스프링 서버**: 방 입·퇴장 처리, 멤버 목록 브로드캐스트, 시그널 메시지 중계 (미디어 데이터는 서버를 통과하지 않음)

### 네임스페이스

* **WebSocket 엔드포인트**: `/ws`
* **App Prefix**: `/app`
* **Topic Prefix**: `/topic`
* **User Prefix**: `/user`

---

## 시그널링 채널 구조

### 브로드캐스트 (구독)

* **멤버 목록**: `/topic/rooms/{roomId}/members`
* **시그널**: `/topic/rooms/{roomId}/signal`
    * (종류: `OFFER`, `ANSWER`, `CANDIDATE`, `READY`, `LEAVE`)

### 개인 큐 (1:1)

* **시스템 메시지**: `/user/queue/system` (예: "Room is full")
* **내 세션ID 통지**: `/user/queue/me` (발신자 결정의 근거)

### 클라이언트 송신 (App Prefix)

* **방 입장**: `/app/rooms/{roomId}/join`
* **시그널 전송**: `/app/rooms/{roomId}/signal`
* **퇴장**: `/app/rooms/{roomId}/leave`

---

##  메시지 스키마 (개념)

### `SignalMessage`

* **`type`**: `OFFER` | `ANSWER` | `CANDIDATE` | `READY` | `LEAVE`
* **`sid`**: 선택 (클라이언트가 자기 식별용으로 포함)
* **`sdp`**: `OFFER`/`ANSWER`일 때 SDP 본문
* **`candidate`**: ICE 후보 정보 (문자열, `sdpMid`, `sdpMLineIndex`)

### `JoinRequest`

* **`roomId`**: 문자열
* **`displayName`**: 선택 (표시명)

---

##  서버 동작 요약

1.  클라이언트가 `/app/rooms/{roomId}/join`으로 입장 요청을 보냅니다.
2.  서버는 `roomId`를 키로 하여 세션 ID 집합으로 방 상태를 관리합니다.
3.  정원(2명) 초과 시 해당 세션에만 `/user/queue/system`으로 안내 메시지를 보냅니다.
4.  입장에 성공하면, 해당 클라이언트에게만 `/user/queue/me`로 자신의 세션 ID를 전달합니다.
5.  방에 있는 모든 참가자에게 `/topic/rooms/{roomId}/members`로 최신 멤버 목록을 브로드캐스트합니다.
6.  인원이 1명에서 2명이 되는 순간, 1회만 `READY` 시그널을 `/topic/rooms/{roomId}/signal`로 브로드캐스트합니다.
7.  이후 클라이언트가 보내는 `OFFER`/`ANSWER`/`CANDIDATE` 시그널은 `/app/rooms/{roomId}/signal`로 수신되어, 서버가 같은 방의 `/topic/rooms/{roomId}/signal` 토픽으로 중계합니다.
8.  클라이언트가 퇴장하거나 소켓 연결이 끊기면, 서버는 해당 세션을 방에서 제거하고 멤버 목록을 갱신합니다. 방이 비게 되면 메모리에서 제거됩니다.

---

##  로컬 테스트 (개념)

1.  서버를 실행한 후, 브라우저 탭 2개(또는 다른 브라우저)에서 동일한 `roomId`로 접속합니다.
2.  한쪽은 발신자, 다른 한쪽은 수신자로 WebRTC 협상을 진행합니다.
3.  양쪽 모두 연결 상태가 `connected`가 되면 상대방의 영상이 화면에 표시됩니다.
4.  **참고**: 브라우저의 자동재생 정책으로 영상이 멈춰 보일 수 있습니다. 이때는 원격 영상 요소를 클릭하여 활성화해주세요.

---

##  트러블슈팅 체크리스트

### 상대 영상 미표시

* 각 탭에서 자신의 카메라 미리보기가 정상적으로 나오는지 확인하세요 (권한/장치 점유 문제).
* 원격 영상 요소를 클릭하여 브라우저 자동재생 정책을 우회해보세요.
* 이전 세션 정보가 꼬였을 수 있으니, 완전히 새로운 `roomId`로 재시도해보세요.

### 연결 불가

* 방화벽이나 NAT 환경 문제를 피하기 위해, 가급적 같은 네트워크/로컬 환경에서 테스트하는 것을 권장합니다 (STUN만 구성된 경우).
* 방 인원이 초과되었는지 확인하세요 (서버로부터 "Room is full" 메시지를 받았는지 확인).
* **OFFER 충돌 (glare)**: 서버가 `/user/queue/me`로 전달한 세션 ID를 기준으로 클라이언트 측에서 발신자를 결정하는 규칙을 구현했는지 확인하세요.

---

##  프로덕션 확장 가이드

### 1) 브로커 / 세션 저장소

* **개발**: 인메모리 `SimpleBroker`
* **운영**: **외부 메시지 브로커** (예: RabbitMQ STOMP, Redis Pub/Sub)로 전환하여 서버를 수평 확장할 수 있도록 구성해야 합니다.
* **방/세션 상태**: **Redis**와 같은 외부 저장소에 `roomId → set(sessionId)` 형태로 저장하여 노드 장애 시에도 상태를 안전하게 보존합니다.

### 2) 인증 / 권한 / 보안

* **TLS (HTTPS/WSS)** 적용은 필수입니다.
* STOMP `CONNECT` 단계 및 `/app/rooms/**` 경로에 **JWT** 또는 세션 기반 인증을 적용합니다.
* **CORS/Origin** 설정을 화이트리스트 방식으로 변경하고, 개발용 `*` 설정은 제거합니다.
* 입력값 검증을 추가합니다 (`roomId` 허용 문자·길이 제한, Rate Limiting 등).

### 3) 관측성 / 운영

* **Spring Actuator**를 도입하고 메트릭(접속/구독/메시지 수, 방/세션 수 등)을 수집하여 모니터링 시스템을 구축합니다.
* **로그 상관키(TraceId)**를 활용하여 문제 추적을 용이하게 하고, 주요 이벤트에 대한 알림을 구성합니다.

### 4) TURN 및 연결성

* 외부망이나 대칭형(Symmetric) NAT 환경의 사용자 간 연결 성공률을 높이기 위해 **TURN 서버** (예: coturn) 구축이 필수적입니다.
* WebRTC ICE 서버 목록에 TURN 서버 정보를 클라이언트 측 설정에 추가해야 합니다.

### 5) 다자간 통화 / 고품질 요구

* P2P 방식은 참가자 수가 증가하면 품질 저하와 리소스 비용이 급증하므로, **SFU (Selective Forwarding Unit)** 도입을 검토해야 합니다.
* **SFU 예시**: Jitsi Videobridge, Janus, mediasoup, LiveKit, Kurento
* **역할 분리**: 스프링 서버(인증/방 관리/토큰 발급) + SFU(미디어 라우팅/녹화/처리) 구조로 확장합니다.

### 6) 배포 / 장애 대응

* **무중단 배포** 시 Graceful Shutdown (세션 드레이닝)을 구현하고, `Health Check`/`Readiness Probe`를 활용합니다.
* 방 상태를 Redis에 저장하면, 인스턴스가 재시작되어도 상태를 복구할 수 있습니다.
* 클라이언트 측에 **자동 재연결 정책** 및 **Exponential Backoff** 로직을 구현하는 것을 권장합니다.
