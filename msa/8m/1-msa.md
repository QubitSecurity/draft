# 필상 시스템 — Day 1 킥오프 (MSA 구조 개요)

1. **날짜**

* 2025-08-05

2. **목표**

* 향후 **4개월** 동안의 개발·구축·운영 로드맵을 **MSA 관점**에서 합의
* 핵심 도메인/마이크로서비스, 데이터/인프라 구조, 비기능(SLO/보안/운영) 기준 확정
* MVP 범위와 **출시 체크리스트**(Go-Live 조건) 정의

---

3. **내용**

### A. 도메인 & 마이크로서비스(초안)

* **Gateway 층**: API Gateway(+Auth/OIDC), Edge(WAF/GSLB)
* **수집/처리**: Ingestor(웹/시스템 로그), EDR Collector, TI Fetcher(외부 TI 연동), Stream Processor(실시간 규칙/ML), Rule/Policy Service
* **사건/운영**: Incident Service, FP/미탐 Adjudicator(DBV360), Ticketing/Workflow, Notifier(Email/SMS/웹훅)
* **데이터/조회**: Search Index(Solr 8/9), Meta-DB(중앙), Region-DB(국가별), Object Storage(증적/아티팩트), Reporting/BI
* **플랫폼/관리**: Config/Feature Flag, Admin, Billing/License, Audit/Compliance(로그 무결성)
* **콘솔/UI**: Analyst Console, Customer Portal

### B. 데이터/인프라 구조(초안)

* **Region Split**: KR/JP 우선(데이터 주권), 중앙 **Meta-DB** + 리전 **Local-DB/Index**
* **메시징**: Event Bus(예: Kafka/Redis Stream) + Dead-Letter Queue
* **Observability**: OpenTelemetry(Trace/Metric/Log), 중앙 대시보드(SLO/SLI)
* **네트워킹**: Nginx/HAProxy, Zero-Trust(서비스 간 mTLS), Egress 통제(데이터 유출 방지)
* **CI/CD**: 멀티 리포/모노레포 혼합, 파이프라인(빌드→보안스캔→테스트→배포), Blue/Green 또는 Canary
* **보안 기준**: 최소권한, 키/비밀 관리(KMS), 데이터 마스킹/토큰화, 감사 추적(증적 유지)

### C. 비기능 요구(SLO 초안)

* **가용성**: 콘솔/APIs 99.9% (월간), **핵심 스트림 파이프라인** 99.95%
* **지연**: 인입→사건 생성 P50 ≤ 5s, P95 ≤ 15s
* **확장성**: 시간당 N회 로그/알림 선형 확장(스케일아웃 기준 정의)
* **준수**: KR/JP 규제(개인정보, 보존주기), 취약점/침해 대응 SLA

### D. 4개월 로드맵(마일스톤)

* **M1 (8월)**: 도메인 모델/ADR, 리포 구조/CI, 스켈레톤 서비스, API 계약(OpenAPI), 인프라 베이스(KR)
* **M2 (9월)**: **MVP 파이프라인**(Ingest→Detect→Incident→Ticket), JP 리전 PoC, Observability 기본 계측
* **M3 (10월)**: WAF/EDR 통합, **DBV360(오탐/미탐 판별)**, TI 연동, 권한/테넌시 하드닝
* **M4 (11월)**: 성능/장애복원/DR 리허설, 운영런북/온콜, 보안 점검, **Go-Live 리뷰 & 승인**

### E. Go-Live 체크리스트(요약)

* SLO 모니터링·알람, DR(리전 단절) 리허설 1회 이상, 데이터 주권 점검(계약/정책/기술)
* 운영런북(티켓/장애/보안), 취약점 스캔/펜테스트 주요 이슈 조치 완료
* 고객 온보딩 경로(계정/빌링/지원) 및 로그 증적 보존 정책 확정

---

4. **예시 설명 (플로우 시나리오: WAF 차단 검증 360 연계)**

* **① Ingestor**가 WAF 로그 수신: `status=406, blocked=1` 사건키 생성
* **② TI Fetcher**가 src_ip 평판/연관 위협 조회 → **Rule/Policy**가 오탐 가능성 스코어링
* **③ Adjudicator**(DBV360)가 동일 사건군 최근 N건과 비교·샘플링(동일 IP의 `200` 정상 응답 사례 포함)
* **④ Incident Service**가 **오탐/미탐/실공격** 판정·증거(원로그, 헤더, 파라미터 시그니처, TI 리포트) 저장
* **⑤ Ticketing**이 정책에 따라 자동 티켓/에스컬레이션, **Notifier**가 담당자/고객에 통지
* **⑥ Console**에서 판정근거(증거 중심)와 대응액션(규칙 조정/차단 해제/추가 탐지)을 원클릭 적용

> 핵심 메시지: **행위·증거 기반**으로 FP(오탐)와 Missed-Attack(미탐)을 분리하여 **정탐률↑, 운영비용↓**.
> 본 시나리오는 KR/JP 리전 분리 저장, 중앙 메타-카탈로그로 **검색/감사/컴플라이언스**를 보장.

---

### 오늘의 산출물(합의 사항)

* 서비스 목록/경계 초안, 데이터/인프라 토폴로지, SLO 초안, 4개월 마일스톤, Go-Live 체크리스트(요약)
