# 필상 시스템 — Day 3 도메인 모델 & API 계약

1. **날짜**

* 2025-08-07

2. **목표**

* 핵심 **바운디드 컨텍스트** 확정 및 엔티티/어그리게잇 정의
* 서비스 간 **API 계약(OpenAPI)**·오류 규칙·버저닝 원칙 합의

---

3. **내용**

### A. 바운디드 컨텍스트(초안)

* **Identity**: 사용자/테넌트/역할/권한, OIDC 클레임 발급
* **Ingest**: 로그 수집 채널/소스/파이프라인 설정
* **Detection**: 룰/모델/판정 결과(정탐/오탐/미탐)
* **Incident**: 사건 수명주기, 증거(Artifacts), 상태 전이
* **Ticketing**: 워크플로우, 에스컬레이션 정책
* **Catalog**: 인덱스/저장소의 메타데이터(코어/보존/암호화)
* **Billing**: 플랜/사용량/청구 항목(개발 단계에선 목업)

### B. 주요 엔티티 & 식별자

* `Tenant(id, name, region, status)`
* `User(id, tenantId, email, roles[])`
* `Pipeline(id, source, status, createdAt)`
* `Rule(id, version, type, enabled)`
* `Incident(id, tenantId, severity, status, createdAt)`
* `Artifact(id, incidentId, type, uri, hash)`
* ID 규칙: **ULID**(시간순 정렬) + `TENANT-PREFIX`(외부 노출 시 마스킹)

### C. API 계약 원칙

* **버저닝**: URL `v{n}` + 스키마 마이그레이션 가이드 포함
* **표준 응답**: `data/meta/error` 3분할, 에러코드 사전 관리
* **페이징**: `cursor` 기반(다중 정렬 지원), `limit ≤ 200`
* **시점 조회**: `asOf`(RFC3339), 일관성 레벨 `consistency=eventual|strong`
* **보안**: OIDC 스코프(`incidents:read`, `rules:write`…), 리소스 소유자 검사

### D. OpenAPI 샘플(발췌)

```yaml
# incident-svc v1
paths:
  /v1/incidents:
    get:
      summary: List incidents
      parameters:
        - in: query; name: tenant_id; required: true
        - in: query; name: cursor
        - in: query; name: severity; schema: { enum: [low,med,high,crit] }
      responses:
        "200": { $ref: "#/components/responses/IncidentList" }
        "401": { $ref: "#/components/responses/AuthError" }
```

### E. 오류 규칙(요약)

* `AUTH_40101` 토큰 만료, `AUTH_40301` 스코프 불충분
* `VAL_422xx` 파라미터 검증 실패(필드 단위 `pointer`)
* `INC_40401` 존재하지 않는 리소스, `INC_40901` 상태 전이 불가

---

4. **예시 설명 (Incident 상태 전이)**

* **열림(open)** → triage → **진행(in_progress)** → **완료(resolved)** 또는 **차단(blocked)**
* 전이 API: `POST /v1/incidents/{id}:transition { action: "resolve", reason, evidence[] }`
* 정책: **역행 금지**, 증거 필수(`evidence.uri/hash`)—감사로그 자동 기록

---
