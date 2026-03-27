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

IMPORTANT: 생성되는 HTML에는 이모지를 절대 사용하지 않는다. 모든 아이콘은 인라인 SVG로 표현한다.

---

## Phase 1: 코드베이스 분석

### Step 1: 프로젝트 타입 감지

다음 파일들을 Glob/Read로 탐색하여 프로젝트 구조를 파악한다:

1. 루트 빌드 파일 확인:
   - `settings.gradle*`, `pom.xml`, `package.json`, `go.mod`, `Cargo.toml`, `*.sln`, `pyproject.toml`, `requirements.txt`
   - 멀티모듈 여부 확인 (settings.gradle의 include, pom.xml의 modules 등)

2. 디렉토리 구조 확인:
   - Glob `*/` 로 1단계 서브디렉토리 목록 확인
   - 각 서브디렉토리에 독립 빌드 파일이 있으면 마이크로서비스/모노레포

3. README.md 확인:
   - 프로젝트 설명, 아키텍처 정보 추출

4. docker-compose 확인:
   - `docker-compose*.yml` 파일에서 서비스 목록과 의존성 추출

판정 기준:
- 서브디렉토리마다 독립 빌드 파일 + 독립 실행 가능 -> `microservices`
- 루트에 단일 빌드 파일 + src/ 구조 -> `monolith`
- 서브디렉토리마다 독립 빌드 파일 + 공유 라이브러리 -> `monorepo`

### Step 2: 서비스/모듈 식별

각 서비스/모듈에 대해 다음 정보를 추출한다:

1. **서비스 이름**: 디렉토리명 또는 빌드 파일의 프로젝트명
2. **포트**:
   - Grep `server.port`, `PORT`, `listen` 등을 설정 파일에서 검색
   - docker-compose.yml의 ports 매핑
3. **기술 스택**:
   - 빌드 파일의 의존성에서 프레임워크 감지 (spring-boot, express, gin, actix, fastapi, django, rails 등)
4. **역할**:
   - 디렉토리명/패키지명에서 추론 (gateway, auth, user, notification 등)
   - README에서 서비스 설명 추출
5. **타입 분류**:
   - `gateway`: 이름에 gateway/proxy 포함 또는 라우팅 설정 존재
   - `frontend`: package.json에 react/next/vue/angular 또는 프론트엔드 프레임워크
   - `common`: 독립 실행 불가한 공유 라이브러리
   - `service`: 그 외 독립 실행 가능한 서비스

### Step 3: 인프라 컴포넌트 감지

설정 파일과 의존성에서 인프라 컴포넌트를 추출한다.

탐색 대상:
- `**/application*.yml`, `**/application*.properties`
- `**/docker-compose*.yml`
- `**/.env*`
- 빌드 파일의 의존성 목록

감지 패턴:

| 컴포넌트 타입 | 감지 키워드 |
|---|---|
| database | mysql, postgresql, postgres, mongodb, oracle, h2, datasource, jdbc, sqlite, mariadb |
| cache | redis, memcached, caffeine, ehcache |
| messageQueue | rabbitmq, kafka, amqp, activemq, pulsar, nats |
| discovery | consul, eureka, zookeeper, nacos |
| monitoring | grafana, prometheus, loki, datadog, alloy, jaeger, zipkin |
| secretManagement | vault, aws-secrets, azure-keyvault |
| cicd | jenkins, github-actions, gitlab-ci, circleci, argocd |
| registry | harbor, ecr, gcr, docker-hub |

각 인프라 컴포넌트에 대해 어떤 서비스가 연결되는지 매핑한다.

### Step 4: API 엔드포인트 및 서비스 간 통신 추적

1. **API 엔드포인트 추출**:
   - Grep으로 컨트롤러/라우터 파일 탐색:
     - Java/Kotlin: `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
     - Node.js: `router.get`, `router.post`, `app.get`, `app.post`
     - Go: `http.HandleFunc`, `r.GET`, `r.POST`, `e.GET`, `e.POST`
     - Python: `@app.route`, `@router.get`, `@api_view`
     - Rust: `#[get`, `#[post`, `.route(`
   - 각 엔드포인트의 HTTP 메서드와 경로 추출

2. **서비스 간 HTTP 통신**:
   - Grep으로 HTTP 클라이언트 호출 탐색:
     - Java: `RestTemplate`, `WebClient`, `@FeignClient`, `RestClient`
     - Node.js: `fetch`, `axios`, `http.request`, `got(`
     - Go: `http.Get`, `http.Post`, `http.NewRequest`
     - Python: `requests.get`, `httpx`, `aiohttp`
   - 호출 대상 URL/서비스명 추출

3. **메시징 통신**:
   - Grep으로 메시지 발행/구독 탐색:
     - RabbitMQ: `@RabbitListener`, `RabbitTemplate`, `amqpTemplate`
     - Kafka: `@KafkaListener`, `KafkaTemplate`, `KafkaProducer`, `KafkaConsumer`
     - 일반: `@EventListener`, `ApplicationEventPublisher`, `EventEmitter`
   - 큐/토픽 이름과 발행자/구독자 매핑

4. **Gateway 라우팅**:
   - API Gateway 설정에서 라우팅 규칙 추출
   - Spring Cloud Gateway: `routes` 설정
   - nginx/envoy: upstream 설정
   - Kong: `services`/`routes` 설정

### Step 5: 데이터 흐름 매핑

위에서 수집한 정보를 종합하여 주요 데이터 흐름을 구성한다:

1. **Gateway 라우팅 기반 흐름**:
   - Client -> Gateway -> 각 서비스 경로를 자동 추출

2. **서비스 간 호출 체인**:
   - HTTP 클라이언트 호출을 추적하여 서비스 간 호출 체인 구성

3. **메시징 흐름**:
   - 발행자 -> 큐/토픽 -> 구독자 경로

4. **DB/캐시 접근**:
   - 각 서비스에서 어떤 인프라에 접근하는지

각 흐름에 이름을 부여하고 (예: "Authentication Flow", "Notification Flow") 경로를 정리한다.

### 분석 결과 데이터 모델

분석 결과를 다음 구조로 정리한다 (실제로 JSON을 생성하지는 않고, 이 구조를 머릿속에 두고 HTML을 생성한다):

```
{
  "projectName": "프로젝트명",
  "projectType": "microservices | monolith | monorepo",
  "services": [
    {
      "name": "서비스명",
      "type": "service | gateway | frontend | common",
      "port": 8080,
      "techStack": ["Spring Boot", "Java"],
      "description": "역할 설명"
    }
  ],
  "infrastructure": [
    {
      "name": "MySQL",
      "type": "database | cache | messageQueue | discovery | monitoring | secretManagement | cicd | registry",
      "connectedServices": ["auth-service", "user-service"]
    }
  ],
  "connections": [
    {
      "from": "api-gateway",
      "to": "auth-service",
      "type": "http | messaging | database",
      "label": "설명"
    }
  ],
  "dataFlows": [
    {
      "name": "Authentication Flow",
      "color": "#FF6B6B",
      "path": ["Client", "api-gateway", "auth-service", "MySQL"]
    }
  ]
}
```

---

## Phase 2: HTML 생성

### 스타일 가이드

#### 색상 팔레트

컴포넌트 유형별 배경색:

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

데이터 흐름 애니메이션 색상 (흐름별 순환):
- 흐름 1: `#FF6B6B`
- 흐름 2: `#4ECDC4`
- 흐름 3: `#45B7D1`
- 흐름 4: `#96CEB4`
- 흐름 5: `#FFEAA7`
- 흐름 6+: 위 색상을 순환 반복

#### 박스 스타일

모든 컴포넌트 박스에 적용하는 CSS:

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

SVG 내부에서는 `<foreignObject>` 대신 순수 SVG로 박스를 그린다:
- `<rect>` 에 `rx="12"` 적용
- `<text>` 로 컴포넌트명 표시, `fill="#FFFFFF"`, `font-family="Segoe UI, system-ui, sans-serif"`, `font-size="14"`, `font-weight="600"`, `text-anchor="middle"`
- 박스 위에 `<filter>` 로 그림자 효과: `<feDropShadow dx="0" dy="2" stdDeviation="4" flood-opacity="0.15"/>`

#### 연결선

SVG `<path>` + `<marker>` 로 화살표:
- 선 색상: `#CBD5E1`
- 선 두께: `2px`
- 화살표 머리: 삼각형 marker

```svg
<defs>
  <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
    <polygon points="0 0, 10 3.5, 0 7" fill="#CBD5E1"/>
  </marker>
</defs>
```

#### 전체 레이아웃

- 배경: `#FFFFFF`
- 전체 너비: `1200px`, 중앙 정렬
- 폰트: `Segoe UI, system-ui, -apple-system, sans-serif`
- 섹션 간 구분: 여백 + 얇은 구분선 (`#E2E8F0`)

#### SVG 아이콘

각 컴포넌트 박스 왼쪽에 유형별 SVG 아이콘을 표시한다. 아이콘은 `24x24` viewBox, 흰색(`#FFFFFF`) 단색, `stroke-width="1.5"`, `fill="none"`, `stroke="currentColor"`.

아이콘을 SVG 내에서 사용할 때는 `<g transform="translate(x,y)">` 로 박스 내 위치에 배치하고, 아이콘 path를 직접 인라인한다.

| 유형 | 아이콘 형태 | SVG 내용 |
|---|---|---|
| client | 모니터 화면 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><rect x="2" y="3" width="20" height="14" rx="2"/><line x1="8" y1="21" x2="16" y2="21"/><line x1="12" y1="17" x2="12" y2="21"/></svg>` |
| gateway | 방패 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><path d="M12 2l8 4v6c0 5.25-3.5 9.74-8 11-4.5-1.26-8-5.75-8-11V6l8-4z"/></svg>` |
| service | 서버 박스 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><rect x="2" y="2" width="20" height="8" rx="2"/><rect x="2" y="14" width="20" height="8" rx="2"/><circle cx="6" cy="6" r="1" fill="#FFF"/><circle cx="6" cy="18" r="1" fill="#FFF"/></svg>` |
| frontend | 브라우저 창 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><rect x="2" y="3" width="20" height="18" rx="2"/><line x1="2" y1="9" x2="22" y2="9"/><circle cx="6" cy="6" r="1" fill="#FFF"/><circle cx="10" cy="6" r="1" fill="#FFF"/><circle cx="14" cy="6" r="1" fill="#FFF"/></svg>` |
| database | 원통형 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M21 5v14c0 1.66-4.03 3-9 3s-9-1.34-9-3V5"/><path d="M21 12c0 1.66-4.03 3-9 3s-9-1.34-9-3"/></svg>` |
| cache | 번개 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>` |
| messageQueue | 편지봉투 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><rect x="2" y="4" width="20" height="16" rx="2"/><path d="M22 4L12 13 2 4"/></svg>` |
| discovery | 나침반 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><circle cx="12" cy="12" r="10"/><polygon points="16.24 7.76 14.12 14.12 7.76 16.24 9.88 9.88" fill="#FFF" opacity="0.3"/></svg>` |
| monitoring | 차트 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><path d="M18 20V10M12 20V4M6 20v-6"/></svg>` |
| secretManagement | 자물쇠 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><rect x="3" y="11" width="18" height="11" rx="2"/><path d="M7 11V7a5 5 0 0110 0v4"/></svg>` |
| cicd | 순환 화살표 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><path d="M21 2v6h-6"/><path d="M3 12a9 9 0 0115-6.7L21 8"/><path d="M3 22v-6h6"/><path d="M21 12a9 9 0 01-15 6.7L3 16"/></svg>` |
| registry | 컨테이너 박스 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#FFF" stroke-width="1.5"><path d="M21 16V8a2 2 0 00-1-1.73l-7-4a2 2 0 00-2 0l-7 4A2 2 0 003 8v8a2 2 0 001 1.73l7 4a2 2 0 002 0l7-4A2 2 0 0021 16z"/><path d="M3.27 6.96L12 12.01l8.73-5.05"/><line x1="12" y1="22.08" x2="12" y2="12"/></svg>` |

### HTML 생성 규칙

#### 파일 구조

단일 HTML 파일로 생성한다. 외부 CDN/라이브러리 의존성은 절대 사용하지 않는다. 순수 HTML + CSS + inline SVG만 사용한다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{프로젝트명} - System Architecture</title>
  <style>
    /* 전체 스타일 인라인 */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
      background: #FFFFFF;
      color: #1a202c;
    }
    .container {
      max-width: 1200px;
      margin: 0 auto;
      padding: 40px 20px;
    }
    header {
      text-align: center;
      margin-bottom: 48px;
    }
    header h1 {
      font-size: 32px;
      font-weight: 700;
      color: #1a202c;
      margin-bottom: 8px;
    }
    .tech-badges {
      display: flex;
      gap: 8px;
      justify-content: center;
      flex-wrap: wrap;
      margin-top: 12px;
    }
    .tech-badge {
      padding: 4px 12px;
      border-radius: 16px;
      background: #F1F5F9;
      color: #475569;
      font-size: 13px;
      font-weight: 500;
    }
    section {
      margin-bottom: 48px;
    }
    section h2 {
      font-size: 24px;
      font-weight: 700;
      color: #1a202c;
      margin-bottom: 24px;
      padding-bottom: 12px;
      border-bottom: 2px solid #E2E8F0;
    }
    .flow-legend {
      display: flex;
      gap: 20px;
      flex-wrap: wrap;
      margin-bottom: 20px;
      padding: 12px 16px;
      background: #F8FAFC;
      border-radius: 8px;
    }
    .legend-item {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 14px;
      color: #475569;
    }
    .legend-dot {
      width: 12px;
      height: 12px;
      border-radius: 50%;
      display: inline-block;
    }
  </style>
</head>
<body>
  <div class="container">
    <!-- 헤더: 프로젝트명 + 프로젝트 타입 + 기술 스택 배지 -->
    <header>
      <h1>{프로젝트명} Architecture</h1>
      <p>{프로젝트 타입}: {서비스 수}개 서비스</p>
      <div class="tech-badges">
        <!-- 감지된 기술 스택을 배지로 나열 -->
        <span class="tech-badge">{기술1}</span>
        <span class="tech-badge">{기술2}</span>
      </div>
    </header>

    <!-- 섹션 1: 시스템 아키텍처 -->
    <section class="architecture">
      <h2>System Architecture</h2>
      <svg width="1200" height="{동적 계산}" viewBox="0 0 1200 {동적 계산}">
        <!-- defs: marker, filter -->
        <defs>
          <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
            <polygon points="0 0, 10 3.5, 0 7" fill="#CBD5E1"/>
          </marker>
          <filter id="shadow">
            <feDropShadow dx="0" dy="2" stdDeviation="4" flood-opacity="0.15"/>
          </filter>
        </defs>

        <!-- 각 레이어별 컴포넌트 박스 + 아이콘 -->
        <!-- 연결선 -->
      </svg>
    </section>

    <!-- 섹션 2: 데이터 흐름 -->
    <section class="data-flow">
      <h2>Data Flow</h2>
      <div class="flow-legend">
        <!-- 흐름별 범례 -->
        <div class="legend-item">
          <span class="legend-dot" style="background: #FF6B6B"></span>
          <span>{흐름 이름}</span>
        </div>
      </div>
      <svg width="1200" height="{동적 계산}" viewBox="0 0 1200 {동적 계산}">
        <!-- 동일한 defs -->
        <!-- 동일한 컴포넌트 배치 (연한 색상 또는 동일) -->
        <!-- 흐름 경로 path + animateMotion dot -->
      </svg>
    </section>
  </div>
</body>
</html>
```

#### 아키텍처 다이어그램 레이아웃

SVG 내부에서 컴포넌트를 계층적으로 배치한다:

```
Layer 0 (y=60):   Client (외부 클라이언트)
Layer 1 (y=180):  Gateway / Load Balancer
Layer 2 (y=300):  Services (가로 배열, 균등 분배)
Layer 3 (y=420):  Infrastructure (DB, Cache, Queue 등 가로 배열)
Layer 4 (y=540):  외부 시스템 (Monitoring, CI/CD, Registry 등)
```

레이아웃 규칙:
- 각 레이어 간 y축 간격: `120px`
- 같은 레이어의 컴포넌트는 x축 기준으로 `1200px` 내에서 균등 배치
- 컴포넌트 박스 크기: 너비 `160px`, 높이 `50px` (텍스트 길이에 따라 조정 가능)
- 컴포넌트가 많은 레이어는 2행으로 분할 (한 행에 최대 6개)
- SVG 전체 높이는 사용된 레이어 수에 맞게 동적 계산

컴포넌트 박스 SVG 구성:
```svg
<!-- 컴포넌트 박스 (예: service 타입) -->
<g transform="translate({x}, {y})" filter="url(#shadow)">
  <!-- 배경 박스 -->
  <rect width="160" height="50" rx="12" fill="#4A90D9"/>
  <!-- 아이콘 (박스 내 왼쪽) -->
  <g transform="translate(14, 15)">
    <!-- 해당 유형의 SVG 아이콘 path -->
  </g>
  <!-- 텍스트 (아이콘 오른쪽) -->
  <text x="95" y="30" fill="#FFFFFF" font-size="14" font-weight="600"
        font-family="Segoe UI, system-ui, sans-serif" text-anchor="middle">{서비스명}</text>
</g>
```

연결선:
```svg
<!-- 직선 연결 -->
<line x1="{시작x}" y1="{시작y}" x2="{끝x}" y2="{끝y}"
      stroke="#CBD5E1" stroke-width="2" marker-end="url(#arrowhead)"/>

<!-- 곡선 연결 (수평 이동이 필요한 경우) -->
<path d="M {시작x} {시작y} C {제어점1x} {제어점1y}, {제어점2x} {제어점2y}, {끝x} {끝y}"
      fill="none" stroke="#CBD5E1" stroke-width="2" marker-end="url(#arrowhead)"/>
```

#### 데이터 흐름 애니메이션

데이터 흐름 섹션에서는 아키텍처와 동일한 컴포넌트 배치를 사용하되, 흐름 경로를 따라 움직이는 원(dot) 애니메이션을 추가한다.

구현 방법:
1. 각 데이터 흐름 경로를 SVG `<path>`로 정의한다
2. 경로 path에는 `stroke-dasharray="8 4"` + `opacity="0.5"` 를 적용하여 점선으로 표시한다
3. 작은 원 (`<circle r="6">`)에 `<animateMotion>`을 적용하여 경로를 따라 이동시킨다
4. 각 흐름마다 고유 색상을 적용한다 (dot fill 색상 + path stroke 색상 동일)
5. `dur="3s"`로 애니메이션 속도를 설정한다
6. `repeatCount="indefinite"`로 무한 반복한다
7. 여러 흐름이 동시에 재생된다

```svg
<!-- 흐름 경로 정의 -->
<path id="flow-1" d="M {시작} L {중간1} L {중간2} L {끝}"
      fill="none" stroke="#FF6B6B" stroke-width="2.5"
      stroke-dasharray="8 4" opacity="0.5"/>

<!-- 움직이는 dot -->
<circle r="6" fill="#FF6B6B" opacity="0.9">
  <animateMotion dur="3s" repeatCount="indefinite">
    <mpath href="#flow-1"/>
  </animateMotion>
</circle>
```

흐름 경로 path의 d 속성:
- 경로상의 각 컴포넌트 중심점을 직선(L)으로 연결한다
- 시작점은 첫 번째 컴포넌트의 하단 중심
- 끝점은 마지막 컴포넌트의 상단 중심
- 중간 컴포넌트는 상단 중심에서 하단 중심으로 통과

#### 범례

데이터 흐름 섹션 상단에 HTML로 범례를 표시한다:

```html
<div class="flow-legend">
  <div class="legend-item">
    <span class="legend-dot" style="background: #FF6B6B"></span>
    <span>Authentication Flow</span>
  </div>
  <div class="legend-item">
    <span class="legend-dot" style="background: #4ECDC4"></span>
    <span>Order Processing Flow</span>
  </div>
  <!-- 흐름 수만큼 반복 -->
</div>
```

---

## 출력

1. 대상 프로젝트 경로에 `docs/` 디렉토리가 없으면 Bash로 `mkdir -p {프로젝트경로}/docs` 실행
2. `{프로젝트경로}/docs/architecture.html` 파일을 Write 도구로 생성
3. 생성 완료 후 메시지 출력: `docs/architecture.html 파일을 생성했습니다. 브라우저에서 열어 확인하세요.`

---

## 주의사항

- 이모지를 절대 사용하지 않는다. 모든 시각적 요소는 SVG 아이콘으로 표현한다.
- 외부 CDN, 라이브러리, 폰트를 로드하지 않는다. 순수 HTML + CSS + inline SVG만 사용한다.
- 반응형 레이아웃은 불필요하다. 1200px 고정 너비를 기준으로 한다.
- SVG 내부에서 `<foreignObject>`는 사용하지 않는다. 순수 SVG 요소만 사용한다.
- 분석 시 실제 존재하는 파일만 읽는다. 추측으로 파일을 만들어내지 않는다.
- 코드베이스에서 확인할 수 없는 컴포넌트는 다이어그램에 포함하지 않는다.
- 서비스가 매우 많은 경우 (10개 초과) 주요 서비스만 포함하고 나머지는 그룹으로 표시한다.
