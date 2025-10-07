# 필상 시스템 — Day 2 개발 환경 설치

1. **날짜**

* 2025-08-06

2. **목표**

* 팀 공통 **개발 표준 환경**을 30분 내 재현 가능하도록 설치
* `make bootstrap → docker compose up` 한 번에 **로컬 전체 스택** 기동

---

3. **내용**

### A. 기본 사양(권장)

* OS: macOS(Apple Silicon/Intel) 또는 Rocky Linux 9
* 필수: Git ≥ 2.40, Docker Desktop/Engine ≥ 24, Docker Compose v2, Make, `curl`, `jq`
* 런타임: Node.js 20 LTS, Python 3.11, Java 17 (Search/Solr용), Go 1.22(선택)
* 편의: `direnv`, `pre-commit`, `asdf`(또는 `mise`)로 버전 고정

```bash
# macOS (brew)
brew install git make jq direnv pre-commit asdf
# Rocky
sudo dnf -y install git make jq direnv python3.11 java-17-openjdk
```

### B. 리포 구조(모노·멀티 혼합)

```
/pil-sang
  /apps
    /gateway         # API Gateway(OIDC 프록시)
    /ingestor        # 로그 수집
    /incident-svc    # 사건/판정
    /ticketing       # 워크플로우
    /console         # 웹 콘솔(Next.js)
  /platform
    /search          # Solr(개발용 단일노드)
    /zookeeper       # Dev 단일 컨테이너
    /meta-db         # Postgres
    /redis           # 큐/캐시
    /minio           # 증적 저장(S3 호환)
    /otel-collector  # OpenTelemetry
    /grafana         # 대시보드
    /nginx           # 로컬 라우팅(80→각 서비스)
  .env.example
  docker-compose.yml
  Makefile
  .pre-commit-config.yaml
  .tool-versions     # asdf 버전 잠금
  .editorconfig
```

### C. 환경 변수(.env)

```dotenv
# 공통
APP_ENV=dev
REGION=kr           # kr|jp
TENANCY=single
# 게이트웨이/OIDC(개발용)
OIDC_ISSUER=http://localhost:8089
OIDC_CLIENT_ID=local-web
OIDC_CLIENT_SECRET=dev-secret
# 데이터
POSTGRES_URL=postgres://pil:pil@localhost:5432/pil
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=pil
S3_SECRET_KEY=pil-secret
# 검색
SOLR_URL=http://localhost:8983/solr
```

> 첫 실행 전 `.env.example`를 복사해 값 채움: `cp .env.example .env`

### D. Docker Compose(개요)

* 컨테이너: `gateway, ingestor, incident-svc, ticketing, console, postgres, redis, minio, zookeeper, solr, otel-collector, grafana, nginx`
* 포트(로컬):

  * Console `http://localhost`
  * Grafana `http://localhost:3000` (admin/admin)
  * Solr `http://localhost:8983`
  * MinIO `http://localhost:9001` (콘솔)

```bash
docker compose pull
docker compose up -d
```

### E. Make 스크립트(핵심 타깃)

```Makefile
bootstrap:
\tcp -n .env || cp .env.example .env
\tpre-commit install || true
\tasdf install || true

up:
\tdocker compose up -d

down:
\tdocker compose down -v

logs:
\tdocker compose logs -f --tail=200

seed:
\t./scripts/dev_seed.sh   # 데모 테넌트/사용자/샘플 로그 주입

lint:
\tnpx eslint apps/console || true && ruff apps || true
```

### F. 품질/보안 훅

* `pre-commit`: `yamllint`, `ruff(파이썬)`, `eslint(웹)`, `gitleaks`(비밀 탐지), `commitlint`(Conventional Commits)
* PR 필수 체크: **빌드·테스트·lint 통과**, `docker compose config` 유효성

```bash
pre-commit install
npx commitlint --init
```

### G. 초기화 스크립트(예)

```bash
# 1) 부트스트랩 & 기동
make bootstrap
make up

# 2) 스키마/코어 생성(검색/DB)
./platform/search/scripts/init-solr.sh         # 코어 생성
./platform/meta-db/scripts/migrate.sh          # DB 마이그레이션

# 3) 데모 데이터
make seed

# 4) 확인
open http://localhost              # 콘솔
open http://localhost:3000         # Grafana
```

---

4. **예시 설명 (신규 개발자 온보딩 20분 체크리스트)**

1) Git 클론 후 `make bootstrap` 실행(훅/버전 잠금 세팅)
2) `.env` 생성 및 기본값 유지(변경 없음)
3) `make up`으로 전체 스택 기동
4) `init-solr.sh`, `migrate.sh`, `make seed` 순서로 초기화
5) 브라우저에서 콘솔 접속 → OIDC Dev 로그인 → 샘플 인입 로그가 **Incident**로 전파되는지 확인
6) Grafana에서 **Ingest→Detect→Incident** 대시 확인(P50 지연, 실패율 지표 노출)

> 목표 기준: 첫날부터 **코드 변경 → 핫리로드 → 콘솔 반영**까지 단일 명령 흐름 확보.
