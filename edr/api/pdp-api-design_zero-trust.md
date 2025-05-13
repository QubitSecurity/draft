## 단말(PLURA Agent)** ↔ **PDP** 간 직접 통신 방식에 대한 설명입니다.

---

## ✅ 전제 정리

* **PDP는 PLURA와 별개의 타사 보안 제품**입니다.
* **PLURA EDR Agent와 PDP는 서로 API를 통해 통신하여 이상 징후를 확인**합니다.
* PLURA Server는 중앙 로그 수집·정책 관리 역할에 집중

---

## ✅ 최적 구현 구조

```plaintext
[PLURA Agent (Windows 단말)]
     │
     │ (1) 단말 보안 상태 분석
     ▼
[PDP (타사 제품)]
     │
     │ (2) API 요청: 상태 보고 또는 이상 여부 질의
     ▼
[정책 판단 / 리소스 접근 판단]
```

* PLURA Agent는 자체 진단 후, **PDP의 API에 보안 상태 정보를 POST**
* PDP는 이를 기반으로 리소스 접근 허용 여부를 판단
* 필요 시 PDP는 **PLURA 서버에도 상태 요청**하거나 **별도로 로그 통합** 가능

---

## 🛡️ 설계 시 핵심 고려사항

### 1. 🔐 **MTLS 적용 (Agent ↔ PDP)**

* 클라이언트 인증서 기반으로 상호 인증
* 키는 Windows 인증서 저장소에 보관, Export 금지 설정

### 2. 🧾 **API 서명 및 위변조 방지**

* Agent가 PDP에 전송하는 데이터는 **Nonce + Timestamp + HMAC** 또는 **JWT 서명** 포함

### 3. 🧠 **Agent 내부 판단 최소화**

* 주요 판단은 **항상 PDP가 수행**
* Agent는 상태만 제공, "허용/차단" 결정은 PDP API 응답에 따름

### 4. ⏱️ **Fallback 구조**

* PDP 응답 지연 시 → 로컬 캐시 정책에 따라 임시 판단 (예: 5분 내 최근 정상 상태 → 임시 허용)

---

## 📦 예시: Agent → PDP API 요청

```http
POST /api/v1/report_status
Authorization: Bearer <token>
Content-Type: application/json

{
  "agent_id": "win10-abcd-1234",
  "timestamp": "2025-05-13T17:00:00Z",
  "status": {
    "malware_detected": false,
    "last_scan_time": "2025-05-13T16:55:00Z",
    "integrity_passed": true
  },
  "signature": "HMAC-SHA256(...)"
}
```

---

## ✅ 결론

> PDP와 PLURA EDR이 **독립된 회사의 시스템**이라면, 다음 구조가 가장 합리적입니다:

| 요소        | 설명                                                |
| --------- | ------------------------------------------------- |
| 통신 방식     | Agent → PDP로 상태 전송, MTLS + HMAC 기반                |
| 권한 분리     | PLURA는 상태만 제공, 판단은 PDP가 전담                        |
| 서버 통신 최적화 | PLURA Agent ↔ PDP만 API 통신, PLURA Server는 보고·분석 전용 |
| 보안 유지     | 인증서 기반 통신 + 서명 + 비추출 키 + API 인증 필수 적용             |

---

다음과 같이 확장할 수 있습니다.

* PLURA ↔ PDP API 명세 예시 (Swagger 기반 초안)
* 메시지 서명 예제 코드 (HMAC, JWT)
* 통합 흐름도 (Mermaid 등)

