# 필상 시스템 — Day 5 접근제어(IAM) & 테넌시 정책

1. **날짜**

* 2025-08-09

2. **목표**

* **인증(OIDC)**·**인가(RBAC+ABAC)**·**테넌시 격리** 원칙 수립
* 관리 콘솔/오픈 API의 **스코프·권한 매트릭스** 확정

---

3. **내용**

### A. 테넌시 모델

* **단일 코드베이스, 데이터 논리 분리**(DB 스키마 분리 + 인덱스 컬렉션 분리)
* 요청 단위 **`tenant_id` 강제**(게이트웨이에서 추출·주입)
* 교차 리전 접근 차단, **메타-카탈로그**만 중앙 공유(인덱스 경로/보존정책)

### B. 인증

* 프로토콜: **OIDC Authorization Code w/ PKCE**
* 토큰: Access(15m), Refresh(8h), **리소스 서버 검증은 mTLS + JWKS**
* 표준 클레임 + 커스텀: `tenant_id`, `roles[]`, `scopes[]`, `region`

### C. 인가

* **RBAC**(역할): `OWNER`, `ADMIN`, `ANALYST`, `READONLY`, `BILLING`
* **ABAC**(속성): `tenant_id == resource.tenant_id`, `region == resource.region`
* 스코프 예시

  * 읽기: `incidents:read`, `search:read`, `rules:read`
  * 쓰기: `incidents:write`, `rules:write`, `notify:send`
  * 관리: `tenant:admin`, `user:manage`

### D. 보안 기준

* 서비스 간 **mTLS**(SPIFFE/SPIRE 가능), 최소권한 서비스 어카운트
* **비밀관리**: KMS, 시크릿 로테이션 90일, 감사로그(접근·변경)
* **레이트 리밋**: 사용자·토큰·IP 레벨(버스트/지속), **리퀘스트 서명**(웹훅)

### E. 데이터 권한 & 마스킹

* 기본 필드 마스킹(PII/IP 정확도), 감사용 원본은 암호화 저장
* 쿼리 레벨 **Row-Level Security**(Postgres RLS) + **인덱스 필터**(Solr 필터쿼리)

### F. 감사·규정 준수

* 모든 보안 이벤트는 `security.audit.v1`로 별도 토픽 전송
* 관리자 행위는 사건과 동일 레벨의 **증거화**(서명된 감사 레코드)

---

4. **예시 설명 (권한 매트릭스 적용 시나리오)**

* **ANALYST**가 `GET /v1/incidents?tenant_id=T1` 호출 → OIDC 토큰의 `tenant_id=T1`, `incidents:read` 확인 → 허용
* 동일 사용자가 `POST /v1/rules` 시도 → 스코프 `rules:write` 부재로 `403 AUTH_40301`
* **ADMIN(T2)**는 T1 리소스 접근 시 **ABAC** 규칙(`tenant_id` 불일치)로 차단
* 웹훅 전송은 `notify:send` + 서명 검증을 통과해야 수신 시스템에서 승인
