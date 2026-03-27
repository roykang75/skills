# architecture-draw

소스 코드를 자동 분석하여 **ByteByteDgo 스타일의 아키텍처 다이어그램 HTML**을 생성하는 Claude Code 스킬입니다.

## 기능

- 코드베이스를 자동 분석하여 프로젝트 구조, 서비스, 인프라, API, 이벤트 흐름을 파악
- ByteByteDgo 인포그래픽 스타일의 HTML 다이어그램 생성 (단일 파일, 외부 의존성 없음)
- 애니메이션 데이터 흐름 (SVG `animateMotion`)
- Hexagonal Architecture, Domain Model, Event Messaging, Swim Lane 흐름 등 다양한 다이어그램 유형 지원

## 생성 예시

| 프로젝트 | 다이어그램 |
|---|---|
| auth-service (Spring Boot, Hexagonal) | Architecture at a Glance, Hexagonal Overview, Domain Model, API Endpoints, Event Messaging, Auth Flow, Security Filter Chain, Tech Stack |
| user-service (Spring Boot, Hexagonal) | Architecture at a Glance, Hexagonal Overview, Domain Model, API Endpoints, Event Messaging, Registration Flow, Security Filter Chain, Tech Stack |

## 사용법

Claude Code에서 슬래시 명령으로 실행합니다.

```bash
# 현재 디렉토리 분석
/architecture-draw

# 특정 프로젝트 경로 지정
/architecture-draw /path/to/project
```

생성된 파일: `{프로젝트}/docs/architecture.html`

브라우저에서 열어 확인할 수 있으며, 오프라인에서도 동작합니다.

## 설치

### 심볼릭 링크 (권장)

```bash
ln -sf /path/to/skills/architecture-draw ~/.claude/skills/architecture-draw
```

### 직접 복사

```bash
cp -r /path/to/skills/architecture-draw ~/.claude/skills/architecture-draw
```

### 설치 확인

```bash
ls -la ~/.claude/skills/architecture-draw/SKILL.md
```

## 삭제

```bash
rm -rf ~/.claude/skills/architecture-draw
```

## 업데이트

`SKILL.md` 파일을 직접 수정하면 됩니다. 새 세션에서 자동 반영되며, 현재 세션에서는 `/reload-plugins`로 즉시 반영할 수 있습니다.

심볼릭 링크로 설치한 경우, 원본 저장소에서 `git pull`하면 자동으로 최신 버전이 적용됩니다.

## 지원 프로젝트 유형

| 유형 | 감지 방식 |
|---|---|
| Spring Boot (Java/Kotlin) | `build.gradle`, `pom.xml`, `@RequestMapping` 등 |
| Node.js (Express, NestJS) | `package.json`, `router.get` 등 |
| Go (Gin, Echo) | `go.mod`, `http.HandleFunc` 등 |
| Python (FastAPI, Django) | `pyproject.toml`, `@app.route` 등 |
| Rust (Actix, Axum) | `Cargo.toml`, `#[get]` 등 |

## 분석 항목

1. **프로젝트 타입 감지** -- monolith, microservices, monorepo
2. **서비스/모듈 식별** -- 이름, 포트, 기술 스택, 역할
3. **인프라 컴포넌트 감지** -- DB, Cache, MQ, Discovery, Monitoring 등
4. **API 엔드포인트 추출** -- REST 컨트롤러에서 HTTP 메서드/경로 파싱
5. **서비스 간 통신 추적** -- HTTP 클라이언트, 메시징(RabbitMQ, Kafka), Gateway 라우팅
6. **데이터 흐름 매핑** -- 주요 비즈니스 흐름을 자동 구성

## 스타일 특성

- 회색 배경 (`#F5F5F5`) 위에 흰색 패널 -- 카드가 부각되는 구조
- 파스텔 색상 박스 + 어두운 텍스트 (`#333`)
- 패널 타이틀에 SVG 아이콘 + 배지 + 설명
- 2열 그리드 레이아웃 (8-11개 패널)
- 코드 요소는 monospace 폰트 (`SF Mono, Fira Code`)
- Swim Lane 스타일 데이터 흐름 (번호 원형 + 색상 코드 화살표)
- SVG `animateMotion` 애니메이션 dot

## 프로젝트 구조

```
architecture-draw/
├── SKILL.md              # 스킬 정의 (분석 절차 + 스타일 가이드 + HTML 생성 규칙)
├── README.md             # 이 문서
├── docs/
│   └── superpowers/
│       ├── specs/        # 설계 문서
│       └── plans/        # 구현 계획
└── images/               # 스크린샷
```

## 참고

- [anthropics/skills](https://github.com/anthropics/skills) -- Claude Code 공식 스킬 저장소
- [Agent Skills Specification](https://agentskills.io/specification) -- 스킬 명세
- [Claude Code Skills 문서](https://code.claude.com/docs/en/skills) -- 공식 문서
