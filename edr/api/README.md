# PLURA-EDR × PDP 연동 문서

이 저장소는 **PLURA-EDR (Agent)** 와 외부 **PDP (Policy Decision Point)** 시스템 간의 연동 설계를 다룹니다.  
Zero Trust Architecture (ZTA) 환경에서, EDR Agent가 상태 정보를 제공하고 PDP가 이를 기반으로 접근 제어 결정을 수행하는 구조를 설명합니다.

---

## 📄 설명

### ZTA 관점에서 API 구조 설계

Zero Trust 관점에서 **PDP의 정책 판단 로직과 API 응답 구조**를 설명합니다.  
Agent로부터 수신된 데이터의 검증, 이상 행위 탐지, 접근 정책 처리 절차를 포함합니다.

- 재전송 방지 (Nonce, Timestamp)
- 서명 검증 및 접근 결정 흐름
- PDP 개발 시 참고용 로직 포함

👉 [pdp-api-design_zero-trust.md](pdp-api-design_zero-trust.md)

---

### 상태 정보 조회

PDP가 보안 상태를 **API 방식으로 PLURA-EDR Agent에 취득(Pull)** 하여  
PDP가 이를 기반으로 접근 허용 또는 차단을 판단하는 전체 흐름을 설명합니다.

- Agent → PDP 방향 통신
- 상태 변경 시만 전송 (이벤트 기반)
- HMAC 서명, MTLS 등 보안 설계 포함

👉 [edr_pdp-integration_status-reporting.md](edr_pdp-integration_status-reporting.md)

---

### PLURA-EDR Agent 상태 정보 WMI 통해 관리

**PLURA-EDR Agent가 Windows WMI를 통해 로컬에서 상태 정보를 제공**하고,  
같은 PC 내에 설치된 PDP Agent가 PowerShell 또는 WMI API를 통해 상태를 조회하는 방식입니다.

- 네트워크 포트 미사용 (로컬 안전 구조)
- Buffer Overflow 등 HTTP 서버 위험 없음
- Windows 10, 11, 서버 환경에 적합

👉 [edr_wmi-status-api.md](edr_wmi-status-api.md)

---

## 🔐 보안 고려 사항

- 단말 내부에 공격자가 존재할 수 있다는 가정 하에 설계
- 통신은 반드시 인증, 서명, 검증을 거쳐야 함
- Agent ↔ PDP 간에는 암묵적인 신뢰 없이 검증 기반 동작 필수

---

## 📌 대상 독자

- PLURA-EDR를 외부 접근제어 시스템과 연동하려는 보안 담당자
- PDP 기능을 개발하거나 연동하는 개발자
- Zero Trust 기반 정책을 Windows 환경에 적용하려는 팀

---

