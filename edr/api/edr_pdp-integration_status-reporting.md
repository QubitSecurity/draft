## ✅ PLURA Agent가 제공해야 할 "상태 정보" 개요

### 📦 상태 정보는 다음과 같이 조화된 요약 데이터(JSON 형태)로 유지합니다:

```json
{
  "agent_id": "win10-abcd-1234",
  "hostname": "DESKTOP-USER1",
  "os_version": "Windows 10 Pro 22H2",
  "status": {
    "malware_detected": false,
    "last_scan_time": "2025-05-13T16:50:00Z",
    "real_time_protection": true,
    "integrity_passed": true,
    "policy_violations": [],
    "last_status_change": "2025-05-13T16:50:00Z"
  }
}
```

---

## ✅ “변경될 때마다” 회신하기 위한 전략

### ① **상태 캐시 구조 유지**

* Agent는 메모리 또는 로컬 파일에 **이전 상태 스냅샷**을 유지
* 주기적으로 현재 상태를 점검한 후, **이전 상태와 비교**

```python
if current_status != cached_status:
    send_to_pdp(current_status)
    cached_status = current_status
```

### ② **상태 변경 감지 기준 예시**

| 항목         | 변경 조건                            |
| ---------- | -------------------------------- |
| 악성코드 감지 여부 | `false → true` 또는 `true → false` |
| 정책 위반 여부   | 위반 항목이 새로 발생하거나 해제됨              |
| 무결성 검증 실패  | 파일 위조나 권한 변경 발생                  |
| 실시간 보호 중지됨 | `real_time_protection: false` 발생 |
| 스캔 실패      | 최근 스캔 시도 실패 기록 추가됨               |

---

## ✅ 회신 방식: 단방향 푸시 or Pull API 호환

### ✅ A. **PDP가 수신하는 API 예시 (Agent → PDP)**

```http
POST /api/v1/status/update
Authorization: Bearer <token>
Content-Type: application/json

{
  "agent_id": "win10-abcd-1234",
  "timestamp": "2025-05-13T16:51:20Z",
  "status": {
    "malware_detected": true,
    "real_time_protection": false,
    "policy_violations": ["USB_BLOCK_DISABLED"],
    "integrity_passed": false
  },
  "signature": "HMAC-SHA256(...)"
}
```

## ✅ B. PDP가 Pull 방식으로 상태 조회하는 API 예시

---

### 📌 엔드포인트 설계

```
GET /api/v1/agents/{agent_id}/status
```

### 🔐 인증 방식

* **MTLS** 또는 **Bearer Token** 인증 필수
* HMAC 또는 JWT 서명 방식도 선택 가능

---

### 📥 요청 예

```http
GET /api/v1/agents/win10-abcd-1234/status
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
Accept: application/json
```

---

### 📤 응답 예 (200 OK)

```json
{
  "agent_id": "win10-abcd-1234",
  "hostname": "DESKTOP-USER1",
  "last_reported_at": "2025-05-13T17:15:00Z",
  "status": {
    "malware_detected": false,
    "real_time_protection": true,
    "integrity_passed": true,
    "policy_violations": [],
    "last_scan_time": "2025-05-13T17:10:00Z"
  },
  "signature": "HMAC-SHA256(...)"
}
```

---

### 📄 오류 응답 예

#### 404 Not Found (Agent 등록 안됨)

```json
{
  "error": "Agent not found",
  "agent_id": "win10-unknown-9999"
}
```

#### 403 Forbidden (인증 실패)

```json
{
  "error": "Unauthorized request"
}
```

---

## ✅ 추가 고려사항

| 항목                        | 설명                                    |
| ------------------------- | ------------------------------------- |
| 응답에 `last_reported_at` 포함 | PDP가 데이터 신선도(예: 10분 이상 경과 여부) 판단 가능   |
| 위변조 방지                    | 응답 본문은 HMAC 또는 JWT 서명 포함하여 확인 가능      |
| 캐싱 전략                     | PDP는 일정 기간 내 조회 결과 캐시 가능 (ex. 5분간 유효) |

---

## 🧩 예시 사용 시나리오

1. 사용자가 내부 시스템에 접근 요청
2. PEP가 PDP에 Agent ID 전달
3. PDP는 PLURA Agent 상태 조회 API 호출
4. 상태 정보 기반으로 접근 허용/차단 결정

---

## 🧩 보완 사항

| 항목              | 설명                                |
| --------------- | --------------------------------- |
| API 전송 실패 시 재시도 | 로컬 큐에 저장하여 재전송 로직 유지 (단절 대비)      |
| 상태 변조 방지        | HMAC 또는 JWT로 서명 후 전송              |
| 이벤트 로깅          | Agent가 변경 상태 전송 시 로컬 로그 남기기 (감사용) |

---

## ✅ 요약

| 구성 요소  | 구현 방식                            |
| ------ | -------------------------------- |
| 상태 유지  | Agent 내 상태 캐시와 현재 상태 비교          |
| 변경 감지  | 악성코드 탐지, 무결성 위반, 실시간 보호 비활성 등 기준 |
| 전송 방식  | 변경 시에만 PDP API로 단방향 푸시           |
| 데이터 형식 | 구조화된 JSON + 서명 포함                |
| 보안     | MTLS + HMAC/JWT 서명으로 위조 방지       |

---

필요하시다면:

* Python 또는 C# 기반 상태 비교 로직 샘플
* API 전송 실패 시 큐 처리 로직
* 상태 변화 전송을 위한 경량 이벤트 처리 설계

도 함께 제공해 드릴 수 있습니다.
