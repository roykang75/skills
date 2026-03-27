# architecture-draw Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 소스 코드를 자동 분석하여 ByteByteDgo 스타일 아키텍처 다이어그램 HTML을 생성하는 Claude Code skill을 만든다.

**Architecture:** 단일 SKILL.md 파일로 구성된 Claude Code skill. 프롬프트가 Claude에게 코드베이스 분석 절차, ByteByteDgo 스타일 규칙, HTML 생성 방법을 지시한다. 외부 스크립트/의존성 없이 Claude의 기존 도구(Glob, Grep, Read, Write)만 사용.

**Tech Stack:** Claude Code Skill (Markdown), HTML, CSS, inline SVG

---

### Task 1: Skill 디렉토리 및 기본 구조 생성

**Files:**
- Create: `SKILL.md`

- [ ] **Step 1: SKILL.md 생성 -- frontmatter + 개요**

```markdown
---
name: architecture-draw
description: 소스 코드를 분석하여 ByteByteDgo 스타일 아키텍처 다이어그램 HTML을 생성. Use when the user wants to generate architecture diagrams, system overview, or data flow visualization from source code.
---

# Architecture Draw

소스 코드를 자동 분석하여 ByteByteDgo 스타일의 시스템 아키텍처 다이어그램과 애니메이션 데이터 흐름을 HTML로 생성한다.

## 실행

- 인자가 있으면 해당 경로를 대상 프로젝트로 사용
- 인자가 없으면 현재 작업 디렉토리를 대상으로 사용
- 출력: `{프로젝트}/docs/architecture.html`
```

- [ ] **Step 2: Skill 심볼릭 링크 등록**

```bash
ln -sf /Users/roy/Workspace/skills/architecture-draw ~/.claude/skills/architecture-draw
```

- [ ] **Step 3: 등록 확인**

```bash
ls -la ~/.claude/skills/architecture-draw
```

Expected: 심볼릭 링크가 `/Users/roy/Workspace/skills/architecture-draw`를 가리킴

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: skill 기본 구조 생성"
```

---

### Task 2: 분석 절차 섹션 작성

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: Phase 1 -- 프로젝트 타입 감지 절차 추가**

SKILL.md에 다음 섹션 추가:

```markdown
## Phase 1: 코드베이스 분석

### Step 1: 프로젝트 타입 감지

다음 파일들을 Glob/Read로 탐색하여 프로젝트 구조를 파악한다:

1. 루트 빌드 파일 확인:
   - `settings.gradle*`, `pom.xml`, `package.json`, `go.mod`, `Cargo.toml`, `*.sln`
   - 멀티모듈 여부 확인 (settings.gradle의 include, pom.xml의 modules 등)

2. 디렉토리 구조 확인:
   - Glob `**/` 로 1단계 서브디렉토리 목록 확인
   - 각 서브디렉토리에 독립 빌드 파일이 있으면 마이크로서비스/모노레포

3. README.md 확인:
   - 프로젝트 설명, 아키텍처 정보 추출

판정 기준:
- 서브디렉토리마다 독립 빌드 파일 + 독립 실행 가능 -> microservices
- 루트에 단일 빌드 파일 + src/ 구조 -> monolith
- 서브디렉토리마다 독립 빌드 파일 + 공유 라이브러리 -> monorepo
```

- [ ] **Step 2: Step 2 -- 서비스/모듈 식별 절차 추가**

```markdown
### Step 2: 서비스/모듈 식별

각 서비스/모듈에 대해 다음 정보를 추출한다:

1. 서비스 이름: 디렉토리명 또는 빌드 파일의 프로젝트명
2. 포트:
   - Grep `server.port`, `PORT`, `listen` 등을 설정 파일에서 검색
   - docker-compose.yml의 ports 매핑
3. 기술 스택:
   - 빌드 파일의 의존성에서 프레임워크 감지 (spring-boot, express, gin, actix 등)
4. 역할:
   - 디렉토리명/패키지명에서 추론 (gateway, auth, user, notification 등)
   - README에서 서비스 설명 추출
5. 타입 분류:
   - "gateway": 이름에 gateway/proxy 포함 또는 라우팅 설정 존재
   - "frontend": package.json에 react/next/vue/angular 또는 프론트엔드 프레임워크
   - "common": 독립 실행 불가한 공유 라이브러리
   - "service": 그 외 독립 실행 가능한 서비스
```

- [ ] **Step 3: Step 3 -- 인프라 컴포넌트 감지 절차 추가**

```markdown
### Step 3: 인프라 컴포넌트 감지

설정 파일과 의존성에서 인프라 컴포넌트를 추출한다:

탐색 대상:
- `**/application*.yml`, `**/application*.properties`
- `**/docker-compose*.yml`
- `**/.env*`
- 빌드 파일의 의존성 목록

감지 패턴:
| 컴포넌트 타입 | 감지 키워드 |
|---|---|
| database | mysql, postgresql, postgres, mongodb, oracle, h2, datasource, jdbc |
| cache | redis, memcached, caffeine, ehcache |
| messageQueue | rabbitmq, kafka, amqp, activemq, pulsar |
| discovery | consul, eureka, zookeeper, nacos |
| monitoring | grafana, prometheus, loki, datadog, alloy |
| secretManagement | vault, aws-secrets, azure-keyvault |
| cicd | jenkins, github-actions, gitlab-ci, circleci |
| registry | harbor, ecr, gcr, docker-hub |

각 인프라 컴포넌트에 대해 어떤 서비스가 연결되는지 매핑한다.
```

- [ ] **Step 4: Step 4 -- API 엔드포인트 및 통신 추적 절차 추가**

```markdown
### Step 4: API 엔드포인트 및 서비스 간 통신 추적

1. API 엔드포인트 추출:
   - Grep으로 컨트롤러/라우터 파일 탐색:
     - Java/Kotlin: `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
     - Node.js: `router.get`, `router.post`, `app.get`, `app.post`
     - Go: `http.HandleFunc`, `r.GET`, `r.POST`
     - Python: `@app.route`, `@router.get`, `@api_view`
   - 각 엔드포인트의 HTTP 메서드와 경로 추출

2. 서비스 간 HTTP 통신:
   - Grep으로 HTTP 클라이언트 호출 탐색:
     - Java: `RestTemplate`, `WebClient`, `@FeignClient`, `RestClient`
     - Node.js: `fetch`, `axios`, `http.request`
     - Go: `http.Get`, `http.Post`, `http.NewRequest`
     - Python: `requests.get`, `httpx`, `aiohttp`
   - 호출 대상 URL/서비스명 추출

3. 메시징 통신:
   - Grep으로 메시지 발행/구독 탐색:
     - RabbitMQ: `@RabbitListener`, `RabbitTemplate`, `amqpTemplate`
     - Kafka: `@KafkaListener`, `KafkaTemplate`
     - 일반: `@EventListener`, `ApplicationEventPublisher`
   - 큐/토픽 이름과 발행자/구독자 매핑

4. Gateway 라우팅:
   - API Gateway 설정에서 라우팅 규칙 추출
   - Spring Cloud Gateway: `routes` 설정
   - nginx/envoy: upstream 설정
```

- [ ] **Step 5: Step 5 -- 데이터 흐름 매핑 절차 추가**

```markdown
### Step 5: 데이터 흐름 매핑

위에서 수집한 정보를 종합하여 주요 데이터 흐름을 구성한다:

1. Gateway 라우팅 기반 흐름:
   - Client -> Gateway -> 각 서비스 경로를 자동 추출

2. 서비스 간 호출 체인:
   - HTTP 클라이언트 호출을 추적하여 서비스 간 호출 체인 구성

3. 메시징 흐름:
   - 발행자 -> 큐/토픽 -> 구독자 경로

4. DB/캐시 접근:
   - 각 서비스에서 어떤 인프라에 접근하는지

각 흐름에 이름을 부여하고 (예: "인증 흐름", "알림 발송 흐름") 경로를 정리한다.
```

- [ ] **Step 6: Commit**

```bash
git add SKILL.md
git commit -m "feat: 코드베이스 분석 절차 추가"
```

---

### Task 3: ByteByteDgo 스타일 가이드 및 SVG 아이콘 섹션 작성

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: 색상 팔레트 및 박스 스타일 규칙 추가**

```markdown
## Phase 2: HTML 생성

### 스타일 가이드

#### 색상 팔레트

컴포넌트 유형별 색상:
| 유형 | 배경색 | 용도 |
|---|---|---|
| gateway | #2C3E6B | API Gateway, Proxy, Load Balancer |
| service | #4A90D9 | 백엔드 서비스 |
| frontend | #7F8C8D | 웹 클라이언트, 모바일 앱 |
| database | #27AE60 | MySQL, PostgreSQL, MongoDB |
| cache | #F39C12 | Redis, Memcached |
| messageQueue | #8E44AD | RabbitMQ, Kafka |
| discovery | #1ABC9C | Consul, Eureka |
| monitoring | #E74C3C | Grafana, Prometheus |
| secretManagement | #E67E22 | Vault |
| cicd | #3498DB | Jenkins, GitHub Actions |
| registry | #16A085 | Harbor, ECR |
| client | #34495E | 외부 클라이언트 |

데이터 흐름 애니메이션 색상 (흐름별 구분):
- 흐름 1: #FF6B6B (빨강 계열)
- 흐름 2: #4ECDC4 (청록 계열)
- 흐름 3: #45B7D1 (하늘 계열)
- 흐름 4: #96CEB4 (연두 계열)
- 흐름 5: #FFEAA7 (노랑 계열)
- 흐름 6+: 위 색상 순환

#### 박스 스타일

```css
.component-box {
  border-radius: 12px;
  padding: 16px 20px;
  color: #FFFFFF;
  font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
  font-size: 14px;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 140px;
  justify-content: center;
}
```

#### 연결선

SVG `<path>` + `<marker>` 로 화살표:
- 선 색상: #CBD5E1 (연한 회색)
- 선 두께: 2px
- 화살표 머리: 삼각형 marker

#### 전체 레이아웃

- 배경: #FFFFFF
- 전체 너비: 1200px, 중앙 정렬
- 상단: 제목 + 프로젝트명 + 기술 스택 요약
- 중단: 시스템 아키텍처 다이어그램 (SVG)
- 하단: 데이터 흐름 애니메이션 (SVG, 흐름별 범례 포함)
- 섹션 간 구분선
```

- [ ] **Step 2: SVG 아이콘 정의 추가**

SKILL.md에 각 컴포넌트 유형별 인라인 SVG 아이콘을 정의한다. 아이콘은 24x24 viewBox 기준, 흰색(#FFFFFF) 단색.

```markdown
#### SVG 아이콘

각 컴포넌트 박스 왼쪽에 유형별 SVG 아이콘을 표시한다. 아이콘은 24x24 viewBox, 흰색 단색, stroke-width: 1.5.

| 유형 | 아이콘 형태 |
|---|---|
| client | 모니터 화면 (rect + 받침대) |
| gateway | 방패 (shield path) |
| service | 서버 박스 (stacked rectangles) |
| frontend | 브라우저 창 (rect + 상단 바 + 3 dots) |
| database | 원통형 (ellipse + rect + ellipse) |
| cache | 번개 (lightning bolt path) |
| messageQueue | 편지봉투 (rect + V 라인) |
| discovery | 나침반 (circle + needle) |
| monitoring | 차트 (bar chart) |
| secretManagement | 자물쇠 (lock) |
| cicd | 순환 화살표 (circular arrows) |
| registry | 박스/컨테이너 (box with whale/container) |

각 아이콘의 구체적인 SVG path는 HTML 생성 시 인라인으로 포함한다.
```

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: ByteByteDgo 스타일 가이드 및 SVG 아이콘 정의 추가"
```

---

### Task 4: HTML 생성 규칙 및 애니메이션 섹션 작성

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: HTML 구조 규칙 추가**

```markdown
### HTML 생성 규칙

#### 파일 구조

단일 HTML 파일로 생성. 외부 CDN/라이브러리 의존성 없음.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{프로젝트명} - System Architecture</title>
  <style>
    /* 전체 스타일 인라인 */
  </style>
</head>
<body>
  <div class="container">
    <!-- 헤더: 프로젝트명, 기술 스택 요약 -->
    <header>...</header>

    <!-- 섹션 1: 시스템 아키텍처 -->
    <section class="architecture">
      <h2>System Architecture</h2>
      <svg><!-- 아키텍처 다이어그램 --></svg>
    </section>

    <!-- 섹션 2: 데이터 흐름 -->
    <section class="data-flow">
      <h2>Data Flow</h2>
      <div class="flow-legend"><!-- 흐름별 범례 --></div>
      <svg><!-- 애니메이션 데이터 흐름 --></svg>
    </section>
  </div>
</body>
</html>
```

#### 아키텍처 다이어그램 레이아웃

SVG 내부에서 컴포넌트를 계층적으로 배치:

```
Layer 0 (최상단): Client
Layer 1: Gateway / Load Balancer
Layer 2: Services (가로 배열)
Layer 3: Infrastructure (DB, Cache, Queue 등 가로 배열)
Layer 4 (최하단): 외부 시스템 (모니터링, CI/CD 등)
```

- 각 레이어는 y축 기준으로 일정 간격 (120px)
- 같은 레이어의 컴포넌트는 x축 기준 균등 배치
- 컴포넌트 간 연결선은 `<path>`로 그리고 `<marker>`로 화살표 표시
- 연결선은 직선 또는 단순한 곡선 (bezier)
```

- [ ] **Step 2: 애니메이션 규칙 추가**

```markdown
#### 데이터 흐름 애니메이션

데이터 흐름 섹션에서는 아키텍처와 동일한 컴포넌트 배치를 사용하되, 흐름 경로를 따라 움직이는 원(dot) 애니메이션을 추가한다.

구현 방법:
1. 각 데이터 흐름 경로를 SVG `<path>`로 정의
2. 작은 원 (`<circle r="6">`)에 `<animateMotion>`을 적용하여 경로를 따라 이동
3. 각 흐름마다 고유 색상 적용 (dot 색상 + path 색상)
4. `dur` 속성으로 애니메이션 속도 조절 (기본 3s)
5. `repeatCount="indefinite"`로 무한 반복
6. 여러 흐름이 동시에 재생됨

```svg
<!-- 흐름 경로 예시 -->
<path id="flow-auth" d="M 600 50 L 600 170 L 300 290 L 300 410"
      fill="none" stroke="#FF6B6B" stroke-width="2" stroke-dasharray="8 4" opacity="0.6"/>

<!-- 움직이는 dot -->
<circle r="6" fill="#FF6B6B">
  <animateMotion dur="3s" repeatCount="indefinite">
    <mpath href="#flow-auth"/>
  </animateMotion>
</circle>
```

#### 범례

데이터 흐름 섹션 상단에 범례를 표시:
- 각 흐름의 색상 dot + 흐름 이름
- 가로로 나열, flex-wrap

```html
<div class="flow-legend">
  <div class="legend-item">
    <span class="legend-dot" style="background: #FF6B6B"></span>
    <span>인증 흐름</span>
  </div>
  <!-- 추가 흐름... -->
</div>
```
```

- [ ] **Step 3: 출력 지시 추가**

```markdown
### 출력

1. 대상 프로젝트 경로에 `docs/` 디렉토리가 없으면 생성
2. `docs/architecture.html` 파일을 Write 도구로 생성
3. 생성 완료 메시지 출력:
   "docs/architecture.html을 생성했습니다. 브라우저에서 열어 확인하세요."
```

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: HTML 생성 규칙 및 애니메이션 섹션 추가"
```

---

### Task 5: kimbap 프로젝트로 skill 테스트

**Files:**
- 없음 (실행 테스트)

- [ ] **Step 1: skill 실행**

```
/architecture-draw /Users/roy/Workspace/kimbap
```

- [ ] **Step 2: 생성된 HTML 확인**

```bash
ls -la /Users/roy/Workspace/kimbap/docs/architecture.html
```

Expected: 파일이 존재하고 크기가 수 KB 이상

- [ ] **Step 3: Playwright로 브라우저에서 열어 확인**

Playwright MCP를 사용하여 생성된 HTML을 브라우저에서 열고 스크린샷을 찍어 시각적으로 검증:
- 아키텍처 다이어그램이 올바르게 렌더링되는지
- 컴포넌트 박스에 SVG 아이콘이 표시되는지
- 데이터 흐름 애니메이션이 동작하는지
- ByteByteDgo 스타일이 적용되었는지

- [ ] **Step 4: 문제점 수정 및 반복**

스크린샷 검토 후 문제가 있으면 SKILL.md를 수정하고 다시 실행하여 확인한다.

- [ ] **Step 5: 최종 Commit**

```bash
git add SKILL.md
git commit -m "feat: skill 테스트 후 최종 조정"
```
