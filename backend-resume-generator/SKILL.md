# Backend Resume Generator

백엔드 프로젝트 코드베이스를 분석하여 개발자 이력서/포트폴리오 항목을 자동 생성하는 스킬입니다.

## When to Use

사용자가 다음과 같은 요청을 할 때 이 스킬을 활성화합니다:
- "이력서 뽑아줘", "포트폴리오 만들어줘"
- "프로젝트 분석해서 경력 정리해줘"
- "resume", "portfolio", "career summary"
- "프로젝트 기술스택 정리해줘"

## How It Works

### Step 1: 프로젝트 탐색

사용자가 지정한 프로젝트 경로(들)에 대해 Explore 에이전트를 병렬로 실행합니다.

각 프로젝트에서 다음을 분석합니다:

**빌드 설정 분석:**
- `build.gradle` / `pom.xml` → 의존성, Java 버전, 프레임워크 버전
- `package.json` → Node.js 프로젝트의 경우
- `requirements.txt` / `pyproject.toml` → Python 프로젝트의 경우

**소스 코드 구조 분석:**
- 디렉토리 구조 → 아키텍처 패턴 식별 (Layered, Hexagonal, DDD 등)
- Controller/Router → API 엔드포인트 목록
- Entity/Model → 도메인 모델
- Service → 핵심 비즈니스 로직
- Config → 인프라 구성 (DB, Cache, MQ 등)

**설정 파일 분석:**
- `application.yml` / `.env` → 사용 기술 (DB, Redis, Kafka 등)
- `docker-compose.yml` → 인프라 구성
- CI/CD 설정 파일

### Step 2: 기술 스택 추출

분석 결과에서 다음 카테고리별로 기술 스택을 분류합니다:

| 카테고리 | 예시 |
|----------|------|
| Language | Java 21, Kotlin, TypeScript |
| Framework | Spring Boot 3.x, NestJS, FastAPI |
| Database | MariaDB, PostgreSQL, MongoDB |
| ORM/Query | JPA, QueryDSL, Prisma |
| Cache | Redis, Caffeine, Ehcache |
| Messaging | Kafka, RabbitMQ, Redis Stream |
| Cloud/Infra | AWS S3, CloudFront, Docker |
| Security | JWT, OAuth2, Spring Security |
| Monitoring | Prometheus, Grafana, ELK |
| Testing | JUnit 5, Mockito, REST Docs |
| CI/CD | GitHub Actions, GitLab CI, Jenkins |
| AI/ML | Spring AI, OpenAI API, LangChain |

### Step 3: 이력서 항목 생성

각 프로젝트에 대해 다음 형식으로 이력서 항목을 생성합니다:

```markdown
## [프로젝트명] — [한줄 요약]
[시작일] ~ [종료일/진행중]

### 프로젝트 개요
- 서비스 설명 (2~3줄)
- 팀 규모, 본인 역할 (알 수 있는 경우)

### 주요 성과 및 기여
- [성과 1: 정량적 수치 포함 권장]
- [성과 2]
- [성과 3]

### 기술적 도전과 해결
- [문제 상황] → [해결 방법] → [결과]

### 기술 스택
Backend: Java 21, Spring Boot 3.x, JPA, QueryDSL
Database: MariaDB, Redis
Infra: AWS S3, CloudFront, Docker
CI/CD: GitLab CI
```

### Step 4: 종합 기술 요약

모든 프로젝트를 종합하여 다음을 생성합니다:

```markdown
## 기술 역량 요약

### Core
- [주력 언어/프레임워크]

### 강점 영역
- [반복적으로 사용한 기술/패턴]

### 아키텍처 경험
- [MSA, 멀티 데이터소스, 이벤트 드리븐 등]
```

## Output Format Options

사용자에게 출력 형식을 물어봅니다:

1. **마크다운** — 기본, 텍스트 기반
2. **노션 페이지** — Notion API를 통해 직접 생성
3. **DOCX** — Word 문서로 출력
4. **PDF** — PDF 문서로 출력

## Analysis Prompt for Explore Agent

Explore 에이전트에게 전달할 프롬프트 템플릿:

```
Thoroughly explore the project at [PROJECT_PATH]. Analyze:
1. Project structure (directories, key files)
2. build.gradle or pom.xml - dependencies, Java/language version, framework version
3. Main application class
4. Key domain entities, controllers, services
5. Tech stack (DB, cache, messaging, cloud services, etc.)
6. Architecture patterns (layered, hexagonal, DDD, etc.)
7. API endpoints and their purposes
8. Security configuration (JWT, OAuth2, etc.)
9. Notable features and design decisions
10. Testing strategy
11. CI/CD pipeline
12. External integrations (3rd party APIs, SDKs)

Return a comprehensive summary covering all above points.
Focus on extractable resume-worthy technical achievements.
```

## Tips for Better Resume Output

- **정량적 수치를 찾아라**: API 엔드포인트 수, 테이블 수, 처리량 등
- **아키텍처 결정을 강조하라**: 왜 그 기술을 선택했는지
- **문제 해결 사례를 추출하라**: 복잡한 설정, 성능 최적화, 보안 처리
- **반복 패턴을 기술 역량으로 전환하라**: 여러 프로젝트에서 공통으로 사용한 기술 = 핵심 역량
