# 필상 시스템 — Day 4 이벤트 스키마 & 스트림 파이프라인

1. **날짜**

* 2025-08-08

2. **목표**

* **이벤트 스키마 v0.1**(인입/판정/사건/알림) 확정
* **멱등성**·순서보장·재처리(DLQ) 원칙 정의 및 PoC

---

3. **내용**

### A. 공통 헤더

```json
{
  "event_id":"01J2Z…", "event_type":"ingest.log.v1",
  "occurred_at":"2025-08-08T01:23:45Z",
  "produced_at":"2025-08-08T01:23:46Z",
  "tenant_id":"TEN_kr_01F…", "trace_id":"8f2c…",
  "source":"waf.kr.edge01", "schema_version":"1"
}
```

### B. 이벤트 타입

* **`ingest.log.v1`**: 원로그 요약(`src_ip`, `dst_host`, `uri`, `status`, `blocked`)
* **`detect.verdict.v1`**: `rule_id`, `verdict(allow|block|review)`, `confidence`, `features[]`
* **`incident.changed.v1`**: `incident_id`, `prev→next`, `severity`, `actor`
* **`notify.requested.v1`**: 채널/수신자/템플릿/중요도

### C. 키·파티셔닝

* **파티션 키**: `tenant_id` (최대 균형)
* **정렬 키**: `occurred_at`(동일 파티션 내 순서)
* **멱등성 키**: `event_id`(저장 전 중복 검사)

### D. 재처리 & DLQ

* 3회 재시도(지수백오프 1s/5s/30s) → **DLQ**로 이동
* DLQ 스키마: 원 이벤트 + `error_type`, `stack`, `first_seen`, `attempts`
* 운영 작업: DLQ 쿼리 → **재주입** API(`POST /v1/dlq/{id}:replay`)

### E. 스키마 유효성

* **JSON Schema** + CI에서 **컨트랙트 테스트**(프로듀서/컨슈머)
* `additionalProperties:false` 기본, 불가피 시 `x-unknown`로 라벨링

### F. 보존/압축

* 실시간 토픽 7일, **컴팩션 토픽**(상태 스냅샷) 30일
* PII 필드 분리·해시화, 원로그는 Object Storage에 **암호화** 저장(URI만 이벤트로)

---

4. **예시 설명 (WAF 차단→판정→사건 생성)**

1) `ingest.log.v1 {status:406,blocked:1}` 수신
2) `detect.verdict.v1 {verdict:"review", confidence:0.62}` 발행(특징량 포함)
3) 규칙 엔진이 **사건키** 생성(`ip+host+uri_sig`) → `incident.changed.v1 {open}`
4) FP 의심(동일 IP의 `200` 정상 다수) 시 **재분류** 이벤트 발행 → 콘솔에 근거 표시
5) 알림 요청(`notify.requested.v1`)이 티켓/메일/웹훅으로 팬아웃

---
