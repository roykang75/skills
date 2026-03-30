---
name: architecture-draw
description: 소스 코드를 분석하여 ByteByteDgo 스타일 아키텍처 다이어그램 HTML을 생성. Use when the user wants to generate architecture diagrams, system overview, or data flow visualization from source code. Supports detail, simple, and poster modes.
---

# Architecture Draw

소스 코드를 자동 분석하여 ByteByteDgo 스타일의 시스템 아키텍처 다이어그램과 애니메이션 데이터 흐름을 HTML로 생성한다.

## 실행

```
/architecture-draw [모드] [경로]
```

- **$0 (모드)**: `detail` (기본값), `simple`, 또는 `poster`
  - `detail`: 8-11개 패널의 상세 아키텍처 문서
  - `simple`: 2-3개 패널의 컴팩트 1페이지 인포그래픽 (ByteByteDgo 단일 이미지 스타일)
  - `poster`: 인포그래픽 포스터 (Lucide 아이콘, S-커브 화살표, 좌표 기반 정밀 레이아웃, Opus 권장)
- **$1 (경로)**: 대상 프로젝트 경로. 없으면 현재 작업 디렉토리 사용
- 경로만 전달하면 (모드 생략) `detail`로 동작

출력 파일:
- `detail` → `{프로젝트}/docs/architecture.html`
- `simple` → `{프로젝트}/docs/architecture-simple.html`
- `poster` → `{프로젝트}/docs/architecture-poster.html`

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

컴포넌트 유형별 색상 (ByteByteGo 파스텔 스타일 — 밝은 배경 + 어두운 텍스트):

| 유형 | 배경색 (Fill) | 테두리색 (Border) | 텍스트색 | 아이콘색 | 용도 |
|---|---|---|---|---|---|
| client | #FFE0C0 | #E6C8A0 | #333333 | #E67E22 | 외부 클라이언트, User |
| gateway | #D4EFFF | #A8D8F0 | #333333 | #2980B9 | API Gateway, Proxy, Load Balancer |
| service | #D4EFFF | #A8D8F0 | #333333 | #4A9EC5 | 백엔드 서비스 |
| frontend | #E8D5F5 | #D0B8E8 | #333333 | #8E44AD | 웹 클라이언트, 모바일 앱 |
| database | #C8F7DC | #A0E0B8 | #333333 | #27AE60 | MySQL, PostgreSQL, MongoDB |
| cache | #FFF9C4 | #E8E0A0 | #333333 | #F5A623 | Redis, Memcached |
| messageQueue | #E8D5F5 | #D0B8E8 | #333333 | #8E44AD | RabbitMQ, Kafka |
| discovery | #C8F7DC | #A0E0B8 | #333333 | #1ABC9C | Consul, Eureka |
| monitoring | #FFD6E0 | #E8B0C0 | #333333 | #E74C3C | Grafana, Prometheus |
| secretManagement | #FFF9C4 | #E8E0A0 | #333333 | #F5A623 | Vault |
| cicd | #D4EFFF | #A8D8F0 | #333333 | #3498DB | Jenkins, GitHub Actions |
| registry | #C8F7DC | #A0E0B8 | #333333 | #16A085 | Harbor, ECR |

데이터 흐름 애니메이션 색상 (흐름별 순환 — 파스텔 배경 위에서 눈에 띄도록 선명한 색상):
- 흐름 1: `#FF6B6B`
- 흐름 2: `#4ECDC4`
- 흐름 3: `#45B7D1`
- 흐름 4: `#96CEB4`
- 흐름 5: `#FFEAA7`
- 흐름 6+: 위 색상을 순환 반복

#### 박스 스타일

모든 컴포넌트 박스에 적용하는 CSS (파스텔 배경 + 어두운 텍스트):

```css
.component-box {
  border-radius: 10px;
  padding: 16px 20px;
  color: #333333;
  font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
  font-size: 14px;
  font-weight: 600;
  border: 1.5px solid; /* 테두리색은 유형별 border color 사용 */
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 140px;
  justify-content: center;
}
```

SVG 내부에서는 `<foreignObject>` 대신 순수 SVG로 박스를 그린다:
- `<rect>` 에 `rx="10"` 적용, `fill`은 유형별 파스텔 배경색, `stroke`는 유형별 테두리색, `stroke-width="1.5"`
- `<text>` 로 컴포넌트명 표시, `fill="#333333"`, `font-family="Segoe UI, system-ui, sans-serif"`, `font-size="14"`, `font-weight="600"`, `text-anchor="middle"`
- 박스 위에 `<filter>` 로 미세한 그림자: `<feDropShadow dx="0" dy="1" stdDeviation="2" flood-opacity="0.1"/>`

#### 연결선

SVG `<path>` + `<marker>` 로 화살표:
- 선 색상: `#888888` (중간 회색)
- 선 두께: `2.5px`
- 화살표 머리: 삼각형 marker, fill `#888888`
- **대시 스타일 (ByteByteDgo 스타일)**: 대부분의 연결선은 dashed 처리한다
  - 컴포넌트 간 연결 화살표: `stroke-dasharray="6 3"` (비동기, REST, 메시징 등)
  - 데이터 흐름 애니메이션 경로: `stroke-dasharray="8 4"`
  - 실선은 직접적인 동기 호출(synchronous direct call)에만 사용한다
- 연결선 위 또는 옆에 **텍스트 라벨**을 반드시 표시한다 (예: "JWT Auth", "AMQP Events", "SQL Queries", "REST API", "gRPC" 등)
- 라벨 스타일: `font-size="11"`, `fill="#666666"`, `font-family="Segoe UI, system-ui, sans-serif"`, 선 중간 지점에 배치

```svg
<defs>
  <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
    <polygon points="0 0, 10 3.5, 0 7" fill="#888888"/>
  </marker>
</defs>

<!-- 연결선 라벨 예시 -->
<text x="{선 중간x}" y="{선 중간y - 8}" font-size="11" fill="#666666"
      font-family="Segoe UI, system-ui, sans-serif" text-anchor="middle">REST API</text>
```

#### 전체 레이아웃

- 배경: `#F5F5F5`
- 전체 너비: `1600px`, 중앙 정렬
- 폰트: `Segoe UI, system-ui, -apple-system, sans-serif`
- 각 섹션을 `.panel` (검정 테두리, 둥근 모서리 12px, 흰색 배경) 로 감싼다
- 패널 내부 서브섹션 구분: 점선 (`stroke-dasharray="6 3"`, `#CCCCCC`)
- 타이틀에 녹색 악센트 바 (header의 `.accent-bar`)

#### SVG 아이콘

각 컴포넌트 박스 왼쪽에 유형별 SVG 아이콘을 표시한다. 아이콘은 `24x24` viewBox, **유형별 고유 색상** (색상 팔레트의 아이콘색 참조), `stroke-width="1.5"`. 아이콘은 단색이 아니라 컴포넌트 특성을 나타내는 고유한 색상을 사용한다.

**아이콘 스타일 가이드 (ByteByteDgo 스타일)**:
- 아이콘은 stroke 전용이 아니라 **fill 색상도 적극 활용**한다 (예: 반투명 fill로 입체감 표현)
- database 아이콘은 3D 원통형으로 그린다: 상단 타원 + 몸통 + 하단 곡선, 그라디언트 효과를 위해 `fill` 에 연한 색상 + `stroke` 에 진한 색상 적용
- 아이콘에 미세한 fill(`opacity="0.15"`)을 추가하여 illustrative한 느낌을 준다
- 단순 outline보다 약간의 두께감과 색채감이 있는 아이콘을 지향한다

아이콘을 SVG 내에서 사용할 때는 `<g transform="translate(x,y)">` 로 박스 내 위치에 배치하고, 아이콘 path를 직접 인라인한다.

#### 오버플로우 방지 규칙 (Overflow Prevention Rules) — CRITICAL

텍스트가 박스 밖으로 넘치는 것을 방지하기 위해 반드시 따라야 하는 규칙:

1. **텍스트 너비 추정**:
   - Monospace font 10.5px: 글자당 ~6.5px
   - Sans-serif font 12px: 글자당 ~7px
   - Sans-serif font 14px: 글자당 ~8px
   - 항상 계산: text_width = char_count * char_width

2. **박스 크기 규칙**:
   - box_width >= text_width + (padding * 2) + icon_width + icon_gap
   - 텍스트와 박스 가장자리 사이 최소 8px 패딩
   - 아이콘 + 텍스트가 같은 박스에 있을 때: icon_width(22px) + gap(8px) 추가

3. **SVG overflow 안전장치**:
   - 모든 SVG 요소: `overflow="visible"` 추가
   - Panel CSS: `overflow: visible` (절대 `overflow: hidden` 사용 금지)

4. **SVG viewBox 크기**:
   - viewBox는 모든 콘텐츠를 담을 만큼 충분히 크게 설정
   - 모든 방향에 20px 여백 추가
   - 콘텐츠가 넘치면 viewBox를 키울 것 — 콘텐츠를 줄이지 말 것

5. **아이콘-텍스트 간격**:
   - SVG 아이콘과 텍스트 사이 간격: 6-8px (그 이상 금지)
   - 텍스트 x 위치 = icon_x + icon_width + 6px

6. **긴 텍스트 처리**:
   - 필요시 클래스명 축약 (예: "AuthCredentialPersistenceAdapter" -> "AuthCredPersistAdapter")
   - 이벤트명 축약 (예: "PasswordChangedEvent" -> "PwdChangedEvent")
   - 텍스트가 여전히 안 맞으면 박스를 넓힐 것 — 9px 이하 폰트 사용 금지

7. **검증**: HTML 생성 후 모든 텍스트 요소를 점검:
   - 텍스트 x 위치 + 추정 텍스트 너비가 부모 rect 범위 안에 있는가?
   - 아니라면 rect 너비를 늘리거나 텍스트 길이를 줄일 것

#### 번호 원형 (Numbered Step Circles)

흐름(Flow) 다이어그램에서 각 단계에 번호 원형을 표시한다. ByteByteDgo 스타일의 빨간 원 + 흰색 숫자:

```svg
<!-- 단계 번호 원형 -->
<circle cx="{x}" cy="{y}" r="12" fill="#FF6B6B" stroke="none"/>
<text x="{x}" y="{y+4}" fill="#FFF" font-size="11" font-weight="700" text-anchor="middle">1</text>
```

- 각 흐름 단계마다 연결선 시작점 근처에 번호 원형을 배치한다
- 번호는 1부터 순서대로 부여한다
- 원형 색상은 해당 흐름의 고유 색상을 사용할 수도 있다 (기본: `#FF6B6B`)

| 유형 | 아이콘 형태 | 아이콘색 | SVG 내용 |
|---|---|---|---|
| client | 모니터 화면 | #E67E22 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#E67E22" stroke-width="1.5"><rect x="2" y="3" width="20" height="14" rx="2"/><line x1="8" y1="21" x2="16" y2="21"/><line x1="12" y1="17" x2="12" y2="21"/></svg>` |
| gateway | 방패 | #2980B9 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#2980B9" stroke-width="1.5"><path d="M12 2l8 4v6c0 5.25-3.5 9.74-8 11-4.5-1.26-8-5.75-8-11V6l8-4z"/></svg>` |
| service | 서버 박스 | #4A9EC5 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#4A9EC5" stroke-width="1.5"><rect x="2" y="2" width="20" height="8" rx="2"/><rect x="2" y="14" width="20" height="8" rx="2"/><circle cx="6" cy="6" r="1" fill="#4A9EC5"/><circle cx="6" cy="18" r="1" fill="#4A9EC5"/></svg>` |
| frontend | 브라우저 창 | #8E44AD | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#8E44AD" stroke-width="1.5"><rect x="2" y="3" width="20" height="18" rx="2"/><line x1="2" y1="9" x2="22" y2="9"/><circle cx="6" cy="6" r="1" fill="#8E44AD"/><circle cx="10" cy="6" r="1" fill="#8E44AD"/><circle cx="14" cy="6" r="1" fill="#8E44AD"/></svg>` |
| database | 원통형 | #27AE60 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#27AE60" stroke-width="1.5"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M21 5v14c0 1.66-4.03 3-9 3s-9-1.34-9-3V5"/><path d="M21 12c0 1.66-4.03 3-9 3s-9-1.34-9-3"/></svg>` |
| cache | 번개 | #F5A623 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#F5A623" stroke-width="1.5"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>` |
| messageQueue | 편지봉투 | #8E44AD | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#8E44AD" stroke-width="1.5"><rect x="2" y="4" width="20" height="16" rx="2"/><path d="M22 4L12 13 2 4"/></svg>` |
| discovery | 나침반 | #1ABC9C | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#1ABC9C" stroke-width="1.5"><circle cx="12" cy="12" r="10"/><polygon points="16.24 7.76 14.12 14.12 7.76 16.24 9.88 9.88" fill="#1ABC9C" opacity="0.3"/></svg>` |
| monitoring | 차트 | #E74C3C | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#E74C3C" stroke-width="1.5"><path d="M18 20V10M12 20V4M6 20v-6"/></svg>` |
| secretManagement | 자물쇠 | #F5A623 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#F5A623" stroke-width="1.5"><rect x="3" y="11" width="18" height="11" rx="2"/><path d="M7 11V7a5 5 0 0110 0v4"/></svg>` |
| cicd | 순환 화살표 | #3498DB | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#3498DB" stroke-width="1.5"><path d="M21 2v6h-6"/><path d="M3 12a9 9 0 0115-6.7L21 8"/><path d="M3 22v-6h6"/><path d="M21 12a9 9 0 01-15 6.7L3 16"/></svg>` |
| registry | 컨테이너 박스 | #16A085 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#16A085" stroke-width="1.5"><path d="M21 16V8a2 2 0 00-1-1.73l-7-4a2 2 0 00-2 0l-7 4A2 2 0 003 8v8a2 2 0 001 1.73l7 4a2 2 0 002 0l7-4A2 2 0 0021 16z"/><path d="M3.27 6.96L12 12.01l8.73-5.05"/><line x1="12" y1="22.08" x2="12" y2="12"/></svg>` |

### SVG vs HTML/CSS 사용 원칙

**HTML/CSS를 우선 사용한다.** SVG는 복잡한 연결선, 애니메이션이 필요한 곳에만 사용한다.

HTML/CSS로 구현하는 요소:
- API 엔드포인트 목록 → flex row (`.api-row`)
- Security Filter Chain → CSS flex (`.filter-chain`)
- Tech Stack 배지 → flex wrap (`.tech-grid`)
- Entity/Data Model 카드 → HTML entity card (`.entity-card`)
- Event Messaging 박스 → CSS grid/flex
- 단순한 컴포넌트 나열 → CSS box 유틸리티 클래스

SVG로 구현하는 요소:
- Architecture at a Glance → 컴포넌트 + 연결선 + 라벨 (SVG)
- Hexagonal Architecture → 계층 구조 + 화살표 (SVG, `flow-svg-container` 감싸기)
- Domain Model 상세 → 필드/메서드/관계 (SVG, `flow-svg-container` 감싸기)
- Data Flow 애니메이션 → `animateMotion` dot (SVG)
- Swim Lane 흐름 → 스윔 레인 + 수평 화살표 (SVG)

SVG 사용 시 규칙:
- 반드시 `<div class="flow-svg-container">` 로 감싼다
- `viewBox`를 사용하고, 고정 width 대신 `style="width:100%;overflow:visible"` 을 적용한다
- 배경 영역 그루핑에는 `fill-opacity="0.3"` 을 적용하여 미묘한 구분을 준다
- 코드 관련 텍스트 (클래스명, 메서드명)는 monospace 폰트 사용: `font-family="SF Mono,Fira Code,Consolas,monospace"`

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
      background: #F5F5F5;
      color: #1a202c;
      --font-mono: 'SF Mono', 'Fira Code', 'Consolas', monospace;
    }
    .container {
      max-width: 1600px;
      margin: 0 auto;
      padding: 40px 48px;
    }
    header {
      text-align: left;
      margin-bottom: 24px;
    }
    /* ByteByteGo 스타일 패널 */
    .panel {
      border: 2px solid #333333;
      border-radius: 12px;
      padding: 24px;
      margin-bottom: 32px;
      background: #FFFFFF;
      overflow: visible;
    }
    .panel-title {
      font-size: 16px;
      font-weight: 700;
      margin-bottom: 16px;
      color: #333;
      padding: 4px 12px;
      padding-bottom: 10px;
      border-bottom: 1.5px dashed #ddd;
      border-radius: 6px;
      display: flex;
      align-items: center;
      gap: 8px;
    }
    /* 패널 그리드: ByteByteDgo 스타일 2열 그리드 배치 */
    .panels-grid {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 24px;
      margin-bottom: 32px;
    }
    .panels-grid .panel {
      margin-bottom: 0;
    }
    /* 전체 너비 패널은 두 열을 모두 차지 */
    .panel-full {
      grid-column: 1 / -1;
    }
    /* 하위 패널: 패널 안에 중첩되는 서브 패널 */
    .sub-panel {
      border: 1.5px solid #DDD;
      border-radius: 8px;
      padding: 16px;
      background: #FAFAFA;
    }
    .sub-panel-title {
      font-size: 14px;
      font-weight: 600;
      margin-bottom: 12px;
      color: #333;
    }
    /* 키워드 하이라이트: 제목에서 핵심 단어를 강조 */
    .highlight {
      background: #C8F7DC;
      padding: 2px 8px;
      border-radius: 4px;
    }
    /* Utility color classes */
    .bg-blue { background: #D4EFFF; }
    .bg-green { background: #C8F7DC; }
    .bg-pink { background: #FFD6E0; }
    .bg-yellow { background: #FFF9C4; }
    .bg-purple { background: #E8D5F5; }
    .bg-orange { background: #FFE0C0; }
    .bg-gray { background: #F0F0F0; }
    /* Badge */
    .badge { display: inline-block; padding: 2px 10px; border-radius: 5px; font-size: 13px; font-weight: 700; color: #333; }
    /* Header */
    .header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px; max-width: 1600px; margin-left: auto; margin-right: auto; }
    .header-title { display: flex; align-items: center; gap: 0; }
    .header-title .accent-bar { width: 6px; height: 48px; background: #2ECC71; border-radius: 3px; margin-right: 14px; }
    .header-title h1 { font-size: 32px; font-weight: 800; letter-spacing: -0.5px; color: #1a1a1a; }
    .brand { font-size: 18px; font-weight: 700; color: #333; display: flex; align-items: center; gap: 6px; }
    /* Flow SVG container */
    .flow-svg-container { width: 100%; overflow: visible; }
    .flow-svg-container svg { width: 100%; height: auto; overflow: visible; }
    /* Entity cards */
    .entity-card { flex: 1; min-width: 280px; border-radius: 10px; overflow: hidden; }
    .entity-header { padding: 8px 14px; font-size: 14px; font-weight: 700; color: #333; display: flex; align-items: center; gap: 6px; }
    .entity-fields { padding: 6px 14px 10px; background: #fff; border: 1.5px solid #ddd; border-top: none; border-radius: 0 0 10px 10px; }
    .entity-field { display: flex; align-items: center; gap: 6px; padding: 3px 0; font-size: 12px; }
    .field-name { font-family: var(--font-mono); font-weight: 600; color: #333; }
    .field-type { color: #888; font-size: 11px; margin-left: auto; }
    /* API endpoint rows */
    .api-group { margin-bottom: 16px; }
    .api-row { display: flex; align-items: center; gap: 8px; padding: 4px 0; font-size: 12.5px; }
    .api-path { font-family: var(--font-mono); font-size: 12px; color: #555; }
    .api-desc { color: #888; font-size: 11.5px; margin-left: auto; }
    .tech-badges, .tech-grid {
      display: flex;
      gap: 18px;
      justify-content: center;
      flex-wrap: wrap;
      margin-top: 12px;
    }
    .tech-badge {
      padding: 14px 22px;
      border-radius: 10px;
      background: #F1F5F9;
      color: #475569;
      font-size: 15px;
      font-weight: 500;
      gap: 10px;
    }
    section {
      margin-bottom: 48px;
    }
    section h2 {
      font-size: 20px;
      font-weight: 700;
      color: #333333;
      margin-bottom: 24px;
      padding-bottom: 0;
      border-bottom: none;
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
    <!-- 헤더: 악센트 바 + 프로젝트명 + 브랜드 -->
    <div class="header">
      <div class="header-title">
        <div class="accent-bar"></div>
        <h1><span class="highlight">{프로젝트명}</span> Architecture</h1>
      </div>
      <div class="brand">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none">
          <rect x="2" y="2" width="20" height="20" rx="4" fill="#2ECC71"/>
          <path d="M7 12h10M12 7v10" stroke="#fff" stroke-width="2.5" stroke-linecap="round"/>
        </svg>
        architecture-draw
      </div>
    </div>
    <header>
      <p>{프로젝트 타입}: {서비스 수}개 서비스</p>
      <div class="tech-badges">
        <!-- 감지된 기술 스택을 배지로 나열 -->
        <span class="tech-badge">{기술1}</span>
        <span class="tech-badge">{기술2}</span>
      </div>
    </header>

    <!-- 섹션들을 그리드로 배치: 독립 개념은 2열 그리드, 복잡한 흐름은 전체 너비 -->
    <!-- 섹션 1: 시스템 아키텍처 (전체 너비 패널) -->
    <section class="architecture">
      <div class="panel panel-full">
        <div class="panel-title">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#4A9EC5" stroke-width="1.5"><rect x="2" y="2" width="20" height="20" rx="4"/><path d="M12 2v20M2 12h20"/></svg>
          <span class="badge bg-blue">System Architecture</span>
          Bird's Eye View
        </div>
        <svg width="1500" height="{동적 계산}" viewBox="0 0 1500 {동적 계산}">
          <!-- defs: marker, filter -->
          <defs>
            <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
              <polygon points="0 0, 10 3.5, 0 7" fill="#888888"/>
            </marker>
            <filter id="shadow">
              <feDropShadow dx="0" dy="1" stdDeviation="2" flood-opacity="0.1"/>
            </filter>
          </defs>

          <!-- 각 레이어별 컴포넌트 박스 + 아이콘 -->
          <!-- 연결선 + 연결선 텍스트 라벨 -->
        </svg>
      </div>
    </section>

    <!-- 섹션 2: 데이터 흐름 (패널로 감싸기) -->
    <section class="data-flow">
      <div class="panel">
        <div class="panel-title">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#FF6B6B" stroke-width="1.5"><path d="M5 12h14M12 5l7 7-7 7"/></svg>
          <span class="badge bg-pink">Data Flow</span>
          Animated Request Lifecycle
        </div>
      <div class="flow-legend">
        <!-- 흐름별 범례 -->
        <div class="legend-item">
          <span class="legend-dot" style="background: #FF6B6B"></span>
          <span>{흐름 이름}</span>
        </div>
      </div>
      <svg width="1500" height="{동적 계산}" viewBox="0 0 1500 {동적 계산}">
        <!-- 동일한 defs -->
        <!-- 동일한 컴포넌트 배치 (연한 색상 또는 동일) -->
        <!-- 흐름 경로 path + animateMotion dot -->
      </svg>
      </div>
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
- 같은 레이어의 컴포넌트는 x축 기준으로 `1500px` 내에서 균등 배치
- 컴포넌트 박스 크기: 너비 `160px`, 높이 `50px` (텍스트 길이에 따라 조정 가능)
- 컴포넌트가 많은 레이어는 2행으로 분할 (한 행에 최대 6개)
- SVG 전체 높이는 사용된 레이어 수에 맞게 동적 계산

컴포넌트 박스 SVG 구성 (파스텔 배경 + 테두리 + 어두운 텍스트 + 컬러 아이콘):
```svg
<!-- 컴포넌트 박스 (예: service 타입) -->
<g transform="translate({x}, {y})" filter="url(#shadow)">
  <!-- 배경 박스: 파스텔 fill + 테두리 -->
  <rect width="160" height="50" rx="10" fill="#D4EFFF" stroke="#A8D8F0" stroke-width="1.5"/>
  <!-- 아이콘 (박스 내 왼쪽, 유형별 컬러) -->
  <g transform="translate(14, 15)">
    <!-- 해당 유형의 SVG 아이콘 path (stroke=유형별 아이콘색) -->
  </g>
  <!-- 텍스트 (아이콘 오른쪽, 어두운 텍스트) -->
  <text x="95" y="30" fill="#333333" font-size="14" font-weight="600"
        font-family="Segoe UI, system-ui, sans-serif" text-anchor="middle">{서비스명}</text>
</g>
```

연결선 (중간 회색 + 텍스트 라벨):
```svg
<!-- 직선 연결 (dashed — 비동기/REST/메시징) -->
<line x1="{시작x}" y1="{시작y}" x2="{끝x}" y2="{끝y}"
      stroke="#888888" stroke-width="2.5" stroke-dasharray="6 3" marker-end="url(#arrowhead)"/>
<!-- 연결선 라벨 -->
<text x="{중간x}" y="{중간y - 8}" font-size="11" fill="#666666"
      font-family="Segoe UI, system-ui, sans-serif" text-anchor="middle">{통신 설명}</text>

<!-- 곡선 연결 (수평 이동이 필요한 경우, dashed) -->
<path d="M {시작x} {시작y} C {제어점1x} {제어점1y}, {제어점2x} {제어점2y}, {끝x} {끝y}"
      fill="none" stroke="#888888" stroke-width="2.5" stroke-dasharray="6 3" marker-end="url(#arrowhead)"/>
<!-- 곡선 연결 라벨 -->
<text x="{곡선 중간x}" y="{곡선 중간y - 8}" font-size="11" fill="#666666"
      font-family="Segoe UI, system-ui, sans-serif" text-anchor="middle">{통신 설명}</text>
```

#### 데이터 흐름 애니메이션

데이터 흐름 섹션에서는 아키텍처와 동일한 컴포넌트 배치를 사용하되, 흐름 경로를 따라 움직이는 원(dot) 애니메이션을 추가한다.

구현 방법:
1. 각 데이터 흐름 경로를 SVG `<path>`로 정의한다
2. 경로 path에는 `stroke-dasharray="8 4"` + `opacity="0.6"` 를 적용하여 점선으로 표시한다 (흐름별 고유 색상 사용)
3. 작은 원 (`<circle r="6">`)에 `<animateMotion>`을 적용하여 경로를 따라 이동시킨다
4. **Z-order (CRITICAL)**: 애니메이션 path와 dot은 반드시 박스(`<rect>`)보다 **먼저** 그린다. SVG는 코드 순서가 z-order이므로, dot을 먼저 그려야 박스 뒤로 지나간다. 단, 번호 ball과 겹치는 화살표는 ball을 화살표 **뒤에** 그려서 ball이 화살표 위에 표시되게 한다.
5. 각 흐름마다 고유 색상을 적용한다 (dot fill 색상 + path stroke 색상 동일)
6. **역방향 화살표 (CRITICAL)**: `orient="auto"` 마커는 line 방향에 맞춰 자동 회전하므로, 역방향 line (x1 > x2)에도 정방향 마커(`M0,0 L8,4 L0,8`)를 사용한다. 별도의 역방향 마커(`swimArrowBack`)를 만들면 이중 반전되어 화살표가 반대로 표시된다.
7. `dur` 속성은 경로 길이에 비례하여 설정한다. 짧은 흐름은 `dur="3s"`, 긴 스윔 레인은 `dur="10~12s"`.
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

#### 인증/데이터 흐름 스타일 (Git Workflow Swim Lane Style)

인증 흐름, 핵심 비즈니스 흐름 등 복잡한 단계별 흐름은 ByteByteDgo의 Git Workflow 다이어그램 스타일의 스윔 레인으로 표현한다:

1. **상단 행: 컴포넌트 스윔 레인**
   - 컴포넌트를 가로로 배치: Client, Service, DB, Cache, MQ (관련 있는 것만)
   - 각 컴포넌트는 파스텔 박스 + 컬러 아이콘 + 설명 텍스트
   - 각 컴포넌트에서 아래로 수직 점선(dotted line) 연장

2. **열 카테고리 헤더**
   - "External" (파랑), "Application" (초록), "Infrastructure" (보라)
   - 관련 열을 아우르는 둥근 사각형 컬러 탭

3. **수직 흐름 + 수평 화살표**
   - 각 단계는 스윔 레인 간 수평 대시 화살표
   - 각 화살표 시작점에 번호 원형 (컬러 배경, 흰색 숫자)
   - 액션 유형별 색상 코드:
     - HTTP 요청: #4A90D9 (파랑)
     - DB 작업: #27AE60 (초록)
     - Redis 작업: #E74C3C (빨강/분홍)
     - 이벤트 발행: #8E44AD (보라)
     - 응답: #F39C12 (주황)

4. **액션 라벨에 흰색 배경 (CRITICAL)**
   - 모든 액션 라벨에는 반드시 흰색 `<rect>`를 뒤에 배치할 것
   - 이렇게 해야 라벨 텍스트가 화살표 선과 겹치지 않음
   - Rect: `fill="white"`, `rx="4"`, `stroke`는 화살표 색상과 동일, `stroke-width="0.8"`, `stroke-dasharray="4 2"`
   - 텍스트는 rect 안에 `text-anchor="middle"`로 중앙 정렬

5. **왼쪽 측면 단계 라벨**
   - 단계들을 그룹으로 묶어 대시 라벨 박스로 표시
   - 예: "Authentication", "Token Generation", "Event Publishing"
   - 대시 테두리, 둥근 모서리, 작은 폰트

6. **분기점**
   - 결정 포인트에 다이아몬드 형태 사용 (예: "2FA?")
   - 다이아몬드에서 두 갈래 경로로 분기

7. **animateMotion 점**
   - 흐름 경로를 따라 움직이는 애니메이션 점 유지
   - `<circle r="6">`, `dur="3s"`, `repeatCount="indefinite"`
   - 각 흐름마다 고유한 밝은 색상

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

### 그리드 레이아웃 가이드

콘텐츠에 여러 독립적인 개념이 있을 때는 2열 그리드(`panels-grid`)로 배치한다. 전체 너비가 필요한 개요 다이어그램이나 복잡한 흐름에는 `panel-full` 클래스를 사용한다.

```html
<div class="panels-grid">
  <div class="panel"><!-- 패널 1: 내부 아키텍처 --></div>
  <div class="panel"><!-- 패널 2: API 엔드포인트 --></div>
  <div class="panel"><!-- 패널 3: 데이터 모델 --></div>
  <div class="panel"><!-- 패널 4: 인증 흐름 --></div>
  <div class="panel panel-full"><!-- 전체 너비: 시스템 개요 --></div>
</div>
```

### 다이어그램 유형 다양화

하나의 문서 안에서 **여러 가지 다이어그램 유형**을 혼합하여 사용한다:

- **흐름 다이어그램 (Flow Diagram)**: 애니메이션 dot이 움직이는 데이터 흐름 표현
- **시퀀스 다이어그램 (Sequence Diagram)**: 세로 타임라인 + 가로 화살표로 서비스 간 호출 순서 표현
- **비교 테이블 (Comparison Table)**: 색상 행이 교대되는 나란한 비교 테이블
- **상태 다이어그램 (State Diagram)**: 노드 + 전환(transition) 화살표로 상태 변화 표현
- **엔티티 관계 다이어그램 (ERD)**: 엔티티 박스 + 관계선으로 데이터 모델 표현

각 패널의 성격에 맞는 다이어그램 유형을 선택한다. 모든 패널을 동일한 유형으로 그리지 않는다.

### 패널 분할 전략 (Panel Splitting Strategy)

콘텐츠가 복잡할 때는 집중된 패널로 분할한다. **"Architecture at a Glance" (간단한 개요)를 항상 첫 번째 패널로 배치**하고, 이후 상세 패널을 나열한다. (Simple -> Detail 흐름)

1. **단일 서비스 분석 시** (8-11개 패널):
   1. Architecture at a Glance (전체 너비) — 조감도, 핵심 컴포넌트와 관계만 간략 표시
   2. Hexagonal/Layered Architecture Overview (전체 너비) — 내부 계층 구조 상세
   3. Domain Model Detail (전체 너비) — 도메인 엔티티 관계
   4. Application Services & Use Cases (반 너비)
   5. Infrastructure Adapters (반 너비)
   6. API Endpoints (반 너비) — 테이블/목록
   7. Data Model (반 너비) — ERD
   8. Event Messaging (전체 너비) — exchange/queue 다이어그램
   9. Authentication/Key Flows (전체 너비, 스윔 레인 스타일)
   10. Security/Filter Chain (전체 너비) — 5개 박스가 수평 공간 필요
   11. Tech Stack (전체 너비)

2. **멀티 서비스 분석 시**: 다음으로 분할:
   - Architecture at a Glance (전체 너비) — 조감도
   - System Overview (전체 너비)
   - 서비스별 요약 (그리드)
   - Data Flow (전체 너비, 애니메이션)
   - Infrastructure (그리드)

**전체 너비 필수 패널**: Security Filter Chain, Tech Stack, Authentication Flows는 항상 `panel-full` (`grid-column: 1 / -1`)로 배치한다. 이들은 수평 공간이 많이 필요하다.

각 패널은 **하나의 개념**에 집중하며, 자체적으로 완결적이어야 한다. ByteByteDgo 스타일처럼 페이지당 **8~11개 패널**을 그리드 레이아웃으로 배치하는 것을 목표로 한다.

### CSS 추가 규칙

```css
/* Security Filter Chain */
.filter-chain { display: flex; flex-wrap: nowrap; justify-content: center; }
.filter-box { padding: 16px 24px; min-width: 160px; font-size: 14px; border-radius: 12px; }
.filter-arrow { margin: 0 12px; }

/* SVG 폰트 크기 기준 */
/* Panel titles: 16px, Section headers: 13px, Box labels: 11-12px, List items: 10-10.5px, Annotations: 9px */
```

### 스타일 레퍼런스

시각적 스타일은 ByteByteDgo의 인포그래픽 다이어그램을 기반으로 한다. 핵심 특성:
- **회색 배경 (`#F5F5F5`)** 위에 **흰색 패널** — 카드가 부각되는 구조
- 파스텔 색상 박스 (어두운 배경에 흰색 텍스트가 아님)
- 모든 박스에 어두운 텍스트 (#333)
- 섹션 그룹핑을 위한 검정 테두리 + 둥근 모서리 패널
- **패널 타이틀에 SVG 아이콘 + 배지 + 설명** (예: `[icon] [Architecture at a Glance] Bird's Eye View`)
- 타이틀에 녹색 악센트 바 (header의 `.accent-bar`)
- 컬러풀한 SVG 아이콘 (흰색 단색이 아님)
- 텍스트 라벨이 있는 대시 회색 화살표 (#888)
- 2열 배치 그리드 레이아웃
- 깔끔하고 교육적인 인포그래픽 느낌
- **코드 요소는 monospace 폰트** (`SF Mono, Fira Code, Consolas`)
- **실제 클래스명, 메서드명, 필드명을 표시** — 추상적 설명이 아닌 구체적 코드 정보
- **정보 밀도**: 각 패널에 실제 코드에서 추출한 구체적 정보를 풍부하게 표시

데이터 흐름에는 Git Workflow 다이어그램 스타일을 따른다:
- 수직 점선이 있는 스윔 레인
- 레인 간 수평 대시 화살표
- 번호 단계 원형
- 흰색 배경 rect가 있는 액션 라벨
- 왼쪽 측면 단계 라벨

---

## 출력

1. 대상 프로젝트 경로에 `docs/` 디렉토리가 없으면 Bash로 `mkdir -p {프로젝트경로}/docs` 실행
2. 모드에 따라 파일 생성:
   - `detail`: `{프로젝트경로}/docs/architecture.html`
   - `simple`: `{프로젝트경로}/docs/architecture-simple.html`
   - `poster`: `{프로젝트경로}/docs/architecture-poster.html`
3. 생성 완료 후 메시지 출력: `docs/architecture[-simple|-poster].html 파일을 생성했습니다. 브라우저에서 열어 확인하세요.`

### simple 모드 규칙

simple 모드는 ByteByteDgo 단일 이미지 인포그래픽 스타일 (images/image.jpg 참조)로 생성한다:

**레이아웃:**
- **흰색 배경** (`background: #fff`) — 포스터/인포그래픽 느낌 (detail 모드의 `#F5F5F5`와 다름)
- `max-width: 1100px` — 컴팩트한 폭
- **2-3개 패널**을 세로로 쌓는다 (전체가 한 화면에 보여야 함)
- 패널 내부는 **좌/우 split** (dashed 구분선) 또는 **상/하 split**으로 분할
- 좌측(SVG 다이어그램) `flex: 1.2` : 우측(텍스트 정보) `flex: 1` 비율 권장

**패널 구성:**
1. **메인 패널** (좌/우 split):
   - 좌측: Architecture Overview SVG (핵심 컴포넌트 + 연결선 + 인프라 + 상태머신)
   - 우측: 서브 섹션들 (dashed 수평 구분선으로 분리) — Hexagonal 구조, API 목록, Event Messaging, Security, Tech Tags
2. **흐름 패널**: 가장 중요한 데이터/비즈니스 흐름 1-2개 (스윔 레인)

**스타일 특성:**
- 패널 테두리: `2.5px solid #222` (detail보다 약간 두꺼움)
- 서브 섹션 제목: 컬러 dot + bold 텍스트 (`.panel-label` 스타일)
- 서브 섹션 구분: dashed 수평선 (`hsplit-divider`)
- 좌/우 구분: dashed 수직선 (`split-divider`)
- 정보 행: 라벨(컬러 bold) + 값(monospace) flex row
- Tech Stack: 작은 태그 (`.tag`) flex wrap
- SVG 아이콘에 **fill 색상 적용** (outline만이 아닌 채워진 아이콘)

**SVG 규칙:**
- viewBox는 콘텐츠에 딱 맞게 설정 (너무 크면 콘텐츠가 작아짐)
- `width:100%` + `overflow:visible` 사용
- 텍스트는 간결하게 (약어 사용 OK)
- 상세 필드/메서드 목록은 생략하고 핵심 구조와 관계만 표현
- 나머지 색상, 아이콘은 detail 모드와 동일

### poster 모드 규칙

poster 모드는 ByteByteDgo 인포그래픽 포스터 스타일로 생성한다. 복잡한 SVG 좌표 계산이 필요하므로 **Opus 모델을 권장**한다.

**레이아웃:**
- 흰색 배경 + 검정 테두리(2.5px) 둥근 패널 (`border-radius: 18px`)
- 패널 상단에 **검정 배경 라벨 칩** (둥근 pill shape)
- 2열 그리드 레이아웃 (주요 패널은 full-width)
- **Inter 폰트** (Google Fonts CDN 사용 허용)

**박스 스타일:**
- 둥근 사각형 (`rx: 12`), 높이 64~66px
- 파스텔 색상 배경 + 진한 테두리:
  - 초록(#d1fae5/#34d399): Backend API
  - 파랑(#dbeafe/#60a5fa): Frontend
  - 보라(#ede9fe/#a78bfa): External Service
  - 청록(#cffafe/#22d3ee): Storage
  - 분홍(#fce7f3/#f472b6): Output
  - 노랑(#fef9c3/#facc15): Config/Prompt/Tool
  - 회색(#f1f5f9/#94a3b8): User/Proxy
  - 주황(#ffedd5/#fb923c): Download/Export
- **Lucide SVG 아이콘** 사용 (CDN: `unpkg.com/lucide@latest` 또는 인라인 SVG path)
- 아이콘 20x20px, stroke-width 1.5~2, 박스 텍스트와 동일 색상

**좌표 계획 (CRITICAL — 반드시 다이어그램 그리기 전에 수행):**

각 박스의 좌표와 연결점을 먼저 계산한다:
```
박스: { x, y, w, h }
중심: cx = x + w/2, cy = y + h/2
연결점:
  top    = (cx, y)
  bottom = (cx, y + h)
  left   = (x, cy)
  right  = (x + w, cy)
```

**화살표 연결 규칙 (MUST FOLLOW):**
1. 시작/끝은 반드시 박스의 edge center (top/bottom/left/right)에 연결
2. cubic bezier의 마지막 좌표가 반드시 target의 edge center여야 함
3. 박스 중심에서 벗어난 연결 금지
4. 화살표가 다른 박스를 관통하면 안 됨
5. 4개 이상 팬아웃 시 시작점 분산 필수

**곡선은 반드시 cubic bezier S-커브:**
- 수직: `M sx,sy C sx,midY ex,midY ex,ey` (midY = (sy+ey)/2)
- 수평: `M sx,sy C midX,sy midX,ey ex,ey` (midX = (sx+ex)/2)
- 꺾이는 곡선 금지 (L-shape, Q 직각 금지)

**팬아웃 S-커브 끝에 수직 L 구간(30px) 필수:**
```
turnY = targetY - 30
path = "M sx,sy C sx,midY tx,midY tx,turnY L tx,targetY"
```

**화살촉 Marker 정의 (MUST INCLUDE):**
```xml
<marker id="ah-색상" viewBox="0 0 10 8" refX="10" refY="4" markerWidth="7" markerHeight="6" orient="auto">
  <path d="M0,0 L10,4 L0,8 Z" fill="색상코드"/>
</marker>
```
- `refX="10"` 필수 (9 이하면 화살촉이 묻힘)
- 사용하는 모든 색상에 대해 개별 marker 정의

**애니메이션:**
- `<animateMotion>` path는 해당 화살표의 path d 속성과 동일해야 함
- 색상으로 데이터 종류 구분 (주황: User, 초록: Backend, 보라: External, 청록: Storage, 분홍: AI)
- `begin` 속성으로 시차 적용 (0.3~0.5s 간격)
- `dur` 2~2.5s, `repeatCount="indefinite"`

**패널 구성:**
1. **System Overview** (full-width): 전체 데이터 흐름 (수직 레이어 구조)
2. **상세 플로우 패널** (2열): 주요 기능별 상세 흐름
3. **Deployment Pipeline** (full-width, 선택): CI/CD 파이프라인
4. 하단 Legend: 색상별 범례

**SVG viewBox 크기:**
- System Overview: 1100x580 이상
- 상세 패널: 520x480 이상
- 행 간격 최소 100px

**셀프 검증 (자동, 최대 3회):**
1. 모든 marker가 `<defs>`에 정의되어 있는지
2. 모든 path에 `marker-end` 속성이 있는지
3. 시작/끝 좌표가 박스 edge center와 정확히 일치하는지
4. animateMotion path가 화살표 path와 동일한지
5. Playwright 스크린샷으로 시각 검증 → 문제 시 자동 수정 반복

---

## 검증 (Playwright Visual QA)

HTML 생성 후 반드시 Playwright MCP로 시각 검증을 수행한다:

1. 로컬 HTTP 서버 실행: `python3 -m http.server 8888` (docs 디렉토리에서)
2. Playwright로 페이지 열기: `browser_navigate` → `http://localhost:8888/{파일명}`
3. 전체 페이지 스크린샷 캡쳐: `browser_take_screenshot` (fullPage: true)
4. 스크린샷에서 다음 항목 검증:
   - 텍스트가 박스 밖으로 넘치지 않는가?
   - 애니메이션 dot 경로가 화살표/연결선과 일치하는가?
   - 박스 간 겹침이 없는가?
   - 레이아웃이 의도대로 배치되었는가?
   - 하단 불필요한 여백이 없는가?
5. 문제 발견 시 HTML을 수정하고 다시 캡쳐하여 확인
6. **최대 5회 반복** — 5회 이내에 해결되지 않으면 현재 상태로 완료하고 발견된 이슈를 사용자에게 보고
7. **최종 스크린샷 저장 (필수)**: 검증 완료 후 최종 화면을 PNG 파일로 저장한다
   - `browser_take_screenshot` (fullPage: true, type: png)
   - 파일명: `{프로젝트경로}/docs/architecture[-simple|-poster]-screenshot.png`
   - 예: `docs/architecture-screenshot.png`, `docs/architecture-simple-screenshot.png`, `docs/architecture-poster-screenshot.png`

---

## 주의사항

- 이모지를 절대 사용하지 않는다. 모든 시각적 요소는 SVG 아이콘으로 표현한다.
- 외부 CDN, 라이브러리, 폰트를 로드하지 않는다. 순수 HTML + CSS + inline SVG만 사용한다.
- 반응형 레이아웃은 불필요하다. 1600px 고정 너비를 기준으로 한다.
- SVG 내부에서 `<foreignObject>`는 사용하지 않는다. 순수 SVG 요소만 사용한다.
- 분석 시 실제 존재하는 파일만 읽는다. 추측으로 파일을 만들어내지 않는다.
- 코드베이스에서 확인할 수 없는 컴포넌트는 다이어그램에 포함하지 않는다.
- 서비스가 매우 많은 경우 (10개 초과) 주요 서비스만 포함하고 나머지는 그룹으로 표시한다.
