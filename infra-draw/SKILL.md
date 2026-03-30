---
name: infra-draw
description: 인프라 설정 파일을 분석하여 ByteByteDgo 스타일 인프라 토폴로지 다이어그램 HTML을 생성. Use when the user wants to generate infrastructure diagrams, network topology, container maps, or deployment visualization from infrastructure configs.
---

# Infra Draw

인프라 설정 파일(docker-compose, Kubernetes, Terraform, nginx 등)을 자동 분석하여 ByteByteDgo 스타일의 인프라 토폴로지 다이어그램과 애니메이션 트래픽 흐름을 HTML로 생성한다.

## 실행

```
/infra-draw [모드] [경로]
```

- **$0 (모드)**: `detail` (기본값) 또는 `simple`
  - `detail`: 8-11개 패널의 상세 인프라 문서
  - `simple`: 2-3개 패널의 컴팩트 1페이지 인포그래픽 (ByteByteDgo 단일 이미지 스타일)
- **$1 (경로)**: 대상 프로젝트 경로. 없으면 현재 작업 디렉토리 사용
- 경로만 전달하면 (모드 생략) `detail`로 동작

출력 파일:
- `detail` → `{프로젝트}/docs/infrastructure.html`
- `simple` → `{프로젝트}/docs/infrastructure-simple.html`

IMPORTANT: 생성되는 HTML에는 이모지를 절대 사용하지 않는다. 모든 아이콘은 인라인 SVG로 표현한다.

---

## Phase 1: 인프라 분석

### Step 1: 인프라 유형 감지

다음 파일들을 Glob/Read로 탐색하여 인프라 구성 방식을 파악한다:

1. 컨테이너 오케스트레이션:
   - `docker-compose*.yml`, `docker-compose*.yaml`
   - `Dockerfile*`, `.dockerignore`
   - `**/k8s/**`, `**/kubernetes/**`, `**/manifests/**`
   - `**/helm/**`, `Chart.yaml`, `values.yaml`

2. IaC (Infrastructure as Code):
   - `**/*.tf`, `**/*.tfvars` (Terraform)
   - `**/cloudformation/**`, `template.yaml` (AWS CloudFormation)
   - `**/pulumi/**`, `Pulumi.yaml`

3. 리버스 프록시 / 로드밸런서:
   - `nginx.conf`, `**/nginx/**`
   - `envoy.yaml`, `**/envoy/**`
   - `Caddyfile`, `traefik.yml`
   - `haproxy.cfg`

4. CI/CD:
   - `.github/workflows/*.yml` (GitHub Actions)
   - `.gitlab-ci.yml`
   - `Jenkinsfile`
   - `.circleci/config.yml`
   - `**/argocd/**`

5. 모니터링 / 관측성:
   - `prometheus.yml`, `**/prometheus/**`
   - `**/grafana/**`, `**/dashboards/**`
   - `**/loki/**`, `**/alloy/**`
   - `filebeat.yml`, `logstash.conf`
   - `otel-collector-config.yaml`

6. 기타 설정:
   - `.env*` (환경 변수)
   - `**/vault/**` (시크릿 관리)
   - `**/consul/**` (서비스 디스커버리)

판정 기준:
- docker-compose만 존재 → `docker-compose`
- k8s manifests 또는 helm chart → `kubernetes`
- terraform 파일 → `terraform`
- 복합 구성 → `hybrid`

### Step 2: 컨테이너/서비스 식별

각 컨테이너/서비스에 대해 다음 정보를 추출한다:

1. **서비스 이름**: docker-compose의 서비스명, k8s deployment/service 이름
2. **이미지**: 사용하는 Docker 이미지 (`image:` 또는 Dockerfile `FROM`)
3. **포트 매핑**: 호스트:컨테이너 포트 (docker-compose `ports:`, k8s `containerPort`/`nodePort`)
4. **네트워크**: 소속 네트워크 (docker-compose `networks:`, k8s namespace)
5. **의존성**: `depends_on`, k8s `initContainers`, healthcheck 의존
6. **볼륨/스토리지**: `volumes:`, k8s `PersistentVolumeClaim`
7. **환경 변수**: `environment:`, `env_file:`, k8s `ConfigMap`/`Secret`
8. **리소스 제한**: `mem_limit`, `cpus`, k8s `resources.limits`
9. **헬스체크**: `healthcheck:`, k8s `livenessProbe`/`readinessProbe`
10. **replicas**: k8s `replicas`, docker-compose `deploy.replicas`

타입 분류:
- `application`: 직접 개발한 서비스 (Dockerfile이 있거나 커스텀 이미지)
- `database`: mysql, postgresql, mongodb, mariadb 등
- `cache`: redis, memcached 등
- `messageQueue`: rabbitmq, kafka, nats 등
- `proxy`: nginx, envoy, traefik, haproxy, caddy
- `monitoring`: prometheus, grafana, loki, jaeger, alloy
- `discovery`: consul, eureka, etcd
- `storage`: minio, s3, nfs
- `cicd`: jenkins, argocd, gitlab-runner

### Step 3: 네트워크 토폴로지 추출

네트워크 구성을 분석한다:

1. **Docker 네트워크**:
   - `docker-compose*.yml`의 `networks:` 정의
   - 각 서비스가 어떤 네트워크에 연결되는지
   - 네트워크 드라이버 (`bridge`, `overlay`, `host`)

2. **Kubernetes 네트워크**:
   - Namespace 구분
   - Service 타입 (`ClusterIP`, `NodePort`, `LoadBalancer`)
   - Ingress 규칙 (호스트, 경로, 백엔드)
   - NetworkPolicy (있는 경우)

3. **리버스 프록시 라우팅**:
   - nginx: `upstream`, `server`, `location` 블록
   - envoy: `clusters`, `routes`, `listeners`
   - traefik: `routers`, `services`, `middlewares`

4. **포트 노출 구조**:
   - 외부 노출 포트 (호스트 바인딩, NodePort, LoadBalancer)
   - 내부 전용 포트 (컨테이너 간 통신)
   - 관리 포트 (모니터링, 디버그)

### Step 4: 스토리지 및 데이터 흐름 추출

1. **볼륨 매핑**:
   - Named volumes, bind mounts, tmpfs
   - k8s PVC/PV, StorageClass
   - 공유 볼륨 (여러 서비스가 동일 볼륨 마운트)

2. **데이터 흐름**:
   - 클라이언트 → 프록시 → 서비스 경로
   - 서비스 간 통신 (환경 변수에서 호스트 참조 추출)
   - 메시징 흐름 (MQ 연결)
   - DB 연결 (datasource URL)

3. **로그/메트릭 흐름**:
   - 로그 수집: filebeat → logstash → elasticsearch 또는 alloy → loki
   - 메트릭 수집: prometheus scrape targets
   - 트레이싱: jaeger/zipkin/otel collector

### Step 5: CI/CD 파이프라인 추출

1. **빌드 단계**: checkout → build → test → docker build → push
2. **배포 단계**: deploy → health check → rollback
3. **환경**: dev, staging, production
4. **트리거**: push, PR, schedule, manual

### 분석 결과 데이터 모델

```
{
  "projectName": "프로젝트명",
  "infraType": "docker-compose | kubernetes | terraform | hybrid",
  "services": [
    {
      "name": "서비스명",
      "type": "application | database | cache | messageQueue | proxy | monitoring | ...",
      "image": "이미지명:태그",
      "ports": [{"host": 8080, "container": 80}],
      "networks": ["frontend", "backend"],
      "volumes": [{"name": "data", "mountPath": "/var/lib/mysql"}],
      "dependsOn": ["mysql", "redis"],
      "replicas": 1
    }
  ],
  "networks": [
    {
      "name": "네트워크명",
      "driver": "bridge",
      "services": ["nginx", "app", "mysql"]
    }
  ],
  "volumes": [
    {
      "name": "볼륨명",
      "driver": "local",
      "mountedBy": ["mysql"]
    }
  ],
  "trafficFlows": [
    {
      "name": "Web Request Flow",
      "color": "#FF6B6B",
      "path": ["Client", "nginx:80", "app:8080", "mysql:3306"]
    }
  ],
  "cicd": {
    "tool": "GitHub Actions",
    "stages": ["build", "test", "deploy"]
  }
}
```

---

## Phase 2: HTML 생성

### 스타일 가이드

architecture-draw와 동일한 ByteByteDgo 파스텔 스타일을 사용한다.

#### 색상 팔레트

인프라 컴포넌트 유형별 색상:

| 유형 | 배경색 | 테두리색 | 아이콘색 | 용도 |
|---|---|---|---|---|
| client | #FFE0C0 | #FFB74D | #E67E22 | 외부 클라이언트, 브라우저 |
| proxy | #D4EFFF | #90CAF9 | #2980B9 | Nginx, Envoy, Traefik, HAProxy |
| application | #D4EFFF | #A8D8F0 | #4A9EC5 | 직접 개발한 서비스 |
| database | #C8F7DC | #81C784 | #27AE60 | MySQL, PostgreSQL, MongoDB |
| cache | #FFF9C4 | #FFD54F | #F5A623 | Redis, Memcached |
| messageQueue | #E8D5F5 | #CE93D8 | #8E44AD | RabbitMQ, Kafka |
| monitoring | #FFD6E0 | #F48FB1 | #E74C3C | Prometheus, Grafana, Loki |
| discovery | #C8F7DC | #81C784 | #1ABC9C | Consul, etcd |
| storage | #FFF9C4 | #FFD54F | #F5A623 | MinIO, S3, NFS |
| cicd | #D4EFFF | #90CAF9 | #3498DB | Jenkins, ArgoCD, GitHub Actions |
| network | #F0F0F0 | #BDBDBD | #757575 | Docker Network, K8s Namespace |

트래픽 흐름 애니메이션 색상:
- 흐름 1: `#FF6B6B`
- 흐름 2: `#4ECDC4`
- 흐름 3: `#45B7D1`
- 흐름 4: `#96CEB4`
- 흐름 5: `#FFEAA7`
- 흐름 6+: 위 색상을 순환 반복

#### SVG 아이콘

| 유형 | 아이콘 형태 | 아이콘색 | SVG 내용 |
|---|---|---|---|
| client | 모니터 화면 | #E67E22 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#E67E22" stroke-width="1.5"><rect x="2" y="3" width="20" height="14" rx="2"/><line x1="8" y1="21" x2="16" y2="21"/><line x1="12" y1="17" x2="12" y2="21"/></svg>` |
| proxy | 방패 | #2980B9 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#2980B9" stroke-width="1.5"><path d="M12 2l8 4v6c0 5.25-3.5 9.74-8 11-4.5-1.26-8-5.75-8-11V6l8-4z"/></svg>` |
| application | 서버 박스 | #4A9EC5 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#4A9EC5" stroke-width="1.5"><rect x="2" y="2" width="20" height="8" rx="2"/><rect x="2" y="14" width="20" height="8" rx="2"/><circle cx="6" cy="6" r="1" fill="#4A9EC5"/><circle cx="6" cy="18" r="1" fill="#4A9EC5"/></svg>` |
| database | 원통형 | #27AE60 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#27AE60" stroke-width="1.5"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M21 5v14c0 1.66-4.03 3-9 3s-9-1.34-9-3V5"/><path d="M21 12c0 1.66-4.03 3-9 3s-9-1.34-9-3"/></svg>` |
| cache | 번개 | #F5A623 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#F5A623" stroke-width="1.5"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>` |
| messageQueue | 편지봉투 | #8E44AD | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#8E44AD" stroke-width="1.5"><rect x="2" y="4" width="20" height="16" rx="2"/><path d="M22 4L12 13 2 4"/></svg>` |
| monitoring | 차트 | #E74C3C | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#E74C3C" stroke-width="1.5"><path d="M18 20V10M12 20V4M6 20v-6"/></svg>` |
| discovery | 나침반 | #1ABC9C | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#1ABC9C" stroke-width="1.5"><circle cx="12" cy="12" r="10"/><polygon points="16.24 7.76 14.12 14.12 7.76 16.24 9.88 9.88" fill="#1ABC9C" opacity="0.3"/></svg>` |
| storage | 컨테이너 박스 | #F5A623 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#F5A623" stroke-width="1.5"><path d="M21 16V8a2 2 0 00-1-1.73l-7-4a2 2 0 00-2 0l-7 4A2 2 0 003 8v8a2 2 0 001 1.73l7 4a2 2 0 002 0l7-4A2 2 0 0021 16z"/><path d="M3.27 6.96L12 12.01l8.73-5.05"/><line x1="12" y1="22.08" x2="12" y2="12"/></svg>` |
| cicd | 순환 화살표 | #3498DB | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#3498DB" stroke-width="1.5"><path d="M21 2v6h-6"/><path d="M3 12a9 9 0 0115-6.7L21 8"/><path d="M3 22v-6h6"/><path d="M21 12a9 9 0 01-15 6.7L3 16"/></svg>` |
| network | 네트워크 | #757575 | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#757575" stroke-width="1.5"><circle cx="12" cy="5" r="3"/><circle cx="5" cy="19" r="3"/><circle cx="19" cy="19" r="3"/><line x1="12" y1="8" x2="5" y2="16"/><line x1="12" y1="8" x2="19" y2="16"/></svg>` |
| docker | 도커 고래 | #2496ED | `<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="#2496ED" stroke-width="1.5"><rect x="1" y="10" width="4" height="4" rx="0.5"/><rect x="6" y="10" width="4" height="4" rx="0.5"/><rect x="11" y="10" width="4" height="4" rx="0.5"/><rect x="6" y="5" width="4" height="4" rx="0.5"/><rect x="11" y="5" width="4" height="4" rx="0.5"/><path d="M0 14c0 0 2 4 12 4s12-2 12-4"/></svg>` |

### SVG vs HTML/CSS 사용 원칙

**HTML/CSS를 우선 사용한다.** SVG는 복잡한 연결선, 애니메이션이 필요한 곳에만 사용한다.

HTML/CSS로 구현하는 요소:
- 포트 매핑 테이블 → flex row
- 환경 변수 목록 → entity card
- Tech Stack 배지 → flex wrap
- 서비스 목록/요약 → CSS card/grid
- 볼륨/스토리지 목록 → entity card
- CI/CD 파이프라인 단계 → CSS flex chain

SVG로 구현하는 요소:
- Infrastructure at a Glance → 서비스 + 연결선 + 라벨
- Network Topology → 네트워크 영역 + 서비스 배치 + 연결선
- Traffic Flow 애니메이션 → `animateMotion` dot (반드시 박스보다 먼저 그려서 박스 뒤로 지나가게 할 것 — SVG z-order). 단, 번호 ball과 겹치는 화살표는 ball을 화살표 뒤에 그려서 ball이 위에 표시되게 한다.
- 역방향 화살표: `orient="auto"` 마커는 line 방향에 맞춰 자동 회전하므로, 역방향 line에도 정방향 마커를 사용한다 (별도 역방향 마커 불필요)
- `dur` 속성: 경로 길이에 비례 (짧은 흐름 3s, 긴 스윔 레인 10~12s)
- 로그/메트릭 수집 흐름 → 화살표 + 경로

SVG 사용 시 규칙:
- 반드시 `<div class="flow-svg-container">` 로 감싼다
- `viewBox`를 사용하고, 고정 width 대신 `style="width:100%;overflow:visible"` 을 적용한다
- 배경 영역 그루핑에는 `fill-opacity="0.3"` 을 적용하여 미묘한 구분을 준다
- 코드 관련 텍스트 (이미지명, 포트)는 monospace 폰트 사용: `font-family="SF Mono,Fira Code,Consolas,monospace"`
- **화살표/연결선이 다른 박스를 관통하면 안 됨 (CRITICAL)**: path가 중간에 있는 박스 위를 지나가지 않도록 경로를 우회한다. 선이 박스 위를 지나가야 하는 경우 반드시 박스 아래로 우회시킨다. SVG z-order로 해결하지 말고 path 자체를 우회 경로로 변경할 것.

### HTML 생성 규칙

#### 파일 구조

단일 HTML 파일로 생성한다. 외부 CDN/라이브러리 의존성은 절대 사용하지 않는다. 순수 HTML + CSS + inline SVG만 사용한다.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{프로젝트명} - Infrastructure</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    :root { --font-mono: 'SF Mono', 'Fira Code', 'Consolas', monospace; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
      background: #F5F5F5;
      color: #333;
      line-height: 1.5;
      padding: 40px 48px;
    }
    .header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px; max-width: 1600px; margin-left: auto; margin-right: auto; }
    .header-title { display: flex; align-items: center; gap: 0; }
    .header-title .accent-bar { width: 6px; height: 48px; background: #2ECC71; border-radius: 3px; margin-right: 14px; }
    .header-title h1 { font-size: 32px; font-weight: 800; letter-spacing: -0.5px; color: #1a1a1a; }
    .header-title h1 .highlight { background: #C8F7DC; padding: 2px 10px; border-radius: 6px; }
    .brand { font-size: 18px; font-weight: 700; color: #333; display: flex; align-items: center; gap: 6px; }
    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; max-width: 1600px; margin: 0 auto; }
    .grid .full-width { grid-column: 1 / -1; }
    .panel { background: #fff; border: 2px solid #333; border-radius: 12px; padding: 24px; overflow: visible; }
    .panel-title {
      font-size: 16px; font-weight: 700; margin-bottom: 18px; padding-bottom: 10px;
      border-bottom: 1.5px dashed #ddd; color: #222;
      display: flex; align-items: center; gap: 8px;
    }
    .badge { display: inline-block; padding: 2px 10px; border-radius: 5px; font-size: 13px; font-weight: 700; color: #333; }
    .bg-blue { background: #D4EFFF; }
    .bg-green { background: #C8F7DC; }
    .bg-pink { background: #FFD6E0; }
    .bg-yellow { background: #FFF9C4; }
    .bg-purple { background: #E8D5F5; }
    .bg-orange { background: #FFE0C0; }
    .bg-gray { background: #F0F0F0; }
    .border-blue { border: 2px solid #90CAF9; }
    .border-green { border: 2px solid #81C784; }
    .border-pink { border: 2px solid #F48FB1; }
    .border-yellow { border: 2px solid #FFD54F; }
    .border-purple { border: 2px solid #CE93D8; }
    .border-orange { border: 2px solid #FFB74D; }
    .box { display: inline-flex; align-items: center; gap: 8px; padding: 8px 14px; border-radius: 8px; font-size: 13px; font-weight: 600; color: #333; }
    .flow-svg-container { width: 100%; overflow: visible; }
    .flow-svg-container svg { width: 100%; height: auto; overflow: visible; }
    .flow-legend { display: flex; gap: 20px; flex-wrap: wrap; margin-bottom: 16px; padding: 10px 14px; background: #F8FAFC; border-radius: 8px; }
    .legend-item { display: flex; align-items: center; gap: 8px; font-size: 13px; color: #475569; }
    .legend-dot { width: 12px; height: 12px; border-radius: 50%; display: inline-block; }
    /* Service Card */
    .svc-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 14px; }
    .svc-card { border-radius: 10px; overflow: hidden; }
    .svc-header { padding: 8px 14px; font-size: 14px; font-weight: 700; color: #333; display: flex; align-items: center; gap: 6px; }
    .svc-body { padding: 8px 14px 10px; background: #fff; border: 1.5px solid #ddd; border-top: none; border-radius: 0 0 10px 10px; font-size: 12px; }
    .svc-row { display: flex; gap: 6px; padding: 2px 0; }
    .svc-label { font-weight: 600; color: #555; min-width: 60px; }
    .svc-value { font-family: var(--font-mono); color: #333; }
    /* Port Table */
    .port-table { width: 100%; border-collapse: collapse; font-size: 12px; }
    .port-table th { background: #F0F0F0; padding: 6px 10px; text-align: left; font-weight: 600; border-bottom: 2px solid #DDD; }
    .port-table td { padding: 5px 10px; border-bottom: 1px solid #EEE; font-family: var(--font-mono); }
    .port-table tr:nth-child(even) td { background: #FAFAFA; }
    .port-ext { color: #E74C3C; font-weight: 700; }
    .port-int { color: #27AE60; }
    /* Pipeline */
    .pipeline { display: flex; align-items: center; justify-content: center; flex-wrap: wrap; gap: 0; padding: 16px 0; }
    .pipeline-step { padding: 12px 20px; border-radius: 10px; font-size: 13px; font-weight: 700; color: #333; text-align: center; min-width: 120px; }
    .pipeline-arrow { margin: 0 8px; color: #888; font-size: 18px; }
    /* Tech Grid */
    .tech-grid { display: flex; flex-wrap: wrap; gap: 14px; justify-content: center; }
    .tech-badge { display: flex; align-items: center; gap: 8px; padding: 12px 20px; border-radius: 10px; font-size: 14px; font-weight: 700; color: #333; }
    /* Network Zone */
    .network-zone { border: 2px dashed; border-radius: 12px; padding: 16px; margin-bottom: 16px; }
    .network-zone-title { font-size: 13px; font-weight: 700; margin-bottom: 10px; display: flex; align-items: center; gap: 6px; }
  </style>
</head>
<body>
  <div class="header">
    <div class="header-title">
      <div class="accent-bar"></div>
      <h1><span class="highlight">{프로젝트명}</span> Infrastructure</h1>
    </div>
    <div class="brand">
      <svg width="24" height="24" viewBox="0 0 24 24" fill="none">
        <rect x="2" y="2" width="20" height="20" rx="4" fill="#2ECC71"/>
        <path d="M7 12h10M12 7v10" stroke="#fff" stroke-width="2.5" stroke-linecap="round"/>
      </svg>
      infra-draw
    </div>
  </div>
  <div class="grid">
    <!-- 패널들 -->
  </div>
</body>
</html>
```

### 패널 분할 전략

1. **Infrastructure at a Glance** (전체 너비) — 조감도: 모든 서비스와 연결 관계
2. **Network Topology** (전체 너비) — 네트워크 영역별 서비스 배치, 포트 노출
3. **Container/Service Map** (전체 너비) — 서비스 카드 그리드 (이미지, 포트, 볼륨, 의존성)
4. **Port Mapping** (반 너비) — 외부/내부 포트 매핑 테이블
5. **Volumes & Storage** (반 너비) — 볼륨 마운트, 공유 볼륨
6. **Environment & Configuration** (반 너비) — 주요 환경 변수, 설정 파일
7. **Health Checks** (반 너비) — 헬스체크 설정, 엔드포인트
8. **Traffic Flow** (전체 너비) — 요청 흐름 애니메이션 (Client → Proxy → App → DB)
9. **Monitoring & Observability** (전체 너비) — 로그/메트릭/트레이싱 수집 흐름
10. **CI/CD Pipeline** (전체 너비) — 빌드/배포 파이프라인 단계
11. **Tech Stack** (전체 너비) — 사용 기술 배지

패널 수는 분석 결과에 따라 조정한다. 해당하는 인프라가 없는 패널은 생략한다.

### 네트워크 토폴로지 레이아웃

네트워크 영역을 dashed border 영역으로 구분하고, 영역 내에 서비스를 배치한다.

```
┌─────────── External (Internet) ───────────┐
│  Client                                    │
└────────────────────────────────────────────┘
              │ :443 / :80
┌─────────── DMZ Network ──────────────────┐
│  Nginx / Traefik (Reverse Proxy)          │
└────────────────────────────────────────────┘
              │ :8080 / :8081
┌─────────── Application Network ───────────┐
│  auth-service    user-service    ...       │
└────────────────────────────────────────────┘
              │ :3306 / :6379 / :5672
┌─────────── Data Network ─────────────────┐
│  MySQL    Redis    RabbitMQ               │
└────────────────────────────────────────────┘
              │ :9090 / :3000
┌─────────── Monitoring Network ────────────┐
│  Prometheus    Grafana    Loki             │
└────────────────────────────────────────────┘
```

SVG에서는 각 네트워크를 `<rect>` 영역 (`fill-opacity="0.15"`, `stroke-dasharray="6 3"`)으로 그리고, 서비스 박스를 영역 안에 배치한다.

### 오버플로우 방지 규칙 — CRITICAL

architecture-draw와 동일한 규칙을 적용한다:

1. 텍스트 너비 추정: sans-serif 14px → 글자당 ~8px
2. box_width >= text_width + padding * 2 + icon_width + gap
3. SVG overflow="visible", Panel overflow: visible
4. viewBox는 콘텐츠보다 충분히 크게 (사방 20px 여백)
5. 긴 이미지명은 축약 (예: `rabbitmq:3.12-management-alpine` → `rabbitmq:3.12-mgmt`)
6. 박스를 넓힐 것 — 9px 이하 폰트 사용 금지

### 스타일 레퍼런스

시각적 스타일은 ByteByteDgo의 인포그래픽 다이어그램을 기반으로 한다. 핵심 특성:
- **회색 배경 (`#F5F5F5`)** 위에 **흰색 패널** — 카드가 부각되는 구조
- 파스텔 색상 박스 + 어두운 텍스트 (#333)
- **패널 타이틀에 SVG 아이콘 + 배지 + 설명**
- 네트워크 영역은 dashed border로 구분
- 컬러풀한 SVG 아이콘 (흰색 단색이 아님)
- 텍스트 라벨이 있는 대시 회색 화살표 (#888)
- 2열 배치 그리드 레이아웃
- **코드 요소는 monospace 폰트** (`SF Mono, Fira Code, Consolas`)
- **실제 이미지명, 포트, 볼륨, 환경 변수를 표시** — 추상적 설명이 아닌 구체적 정보
- **정보 밀도**: 각 패널에 실제 설정에서 추출한 구체적 정보를 풍부하게 표시

---

## 출력

1. 대상 프로젝트 경로에 `docs/` 디렉토리가 없으면 Bash로 `mkdir -p {프로젝트경로}/docs` 실행
2. 모드에 따라 파일 생성:
   - `detail`: `{프로젝트경로}/docs/infrastructure.html`
   - `simple`: `{프로젝트경로}/docs/infrastructure-simple.html`
3. 생성 완료 후 메시지 출력: `docs/infrastructure[-simple].html 파일을 생성했습니다. 브라우저에서 열어 확인하세요.`

### simple 모드 규칙

simple 모드는 ByteByteDgo 단일 이미지 인포그래픽 스타일로 생성한다:

**레이아웃:**
- **흰색 배경** (`background: #fff`) — 포스터/인포그래픽 느낌 (detail 모드의 `#F5F5F5`와 다름)
- `max-width: 1100px` — 컴팩트한 폭
- **2-3개 패널**을 세로로 쌓는다 (전체가 한 화면에 보여야 함)
- 패널 내부는 **좌/우 split** (dashed 구분선) 또는 **상/하 split**으로 분할
- 좌측(SVG 다이어그램) `flex: 1.2` : 우측(텍스트 정보) `flex: 1` 비율 권장

**패널 구성:**
1. **메인 패널** (좌/우 split):
   - 좌측: Infrastructure Overview SVG (네트워크 토폴로지 + 서비스 + 연결선)
   - 우측: 서브 섹션들 (dashed 수평 구분선으로 분리) — Container Map, Port Mapping, Volume/Storage, Tech Tags
2. **흐름 패널**: 트래픽 흐름 또는 데이터 경로 1-2개 (스윔 레인)

**스타일 특성:**
- 패널 테두리: `2.5px solid #222` (detail보다 약간 두꺼움)
- 서브 섹션 제목: 컬러 dot + bold 텍스트
- 서브 섹션 구분: dashed 수평선
- 좌/우 구분: dashed 수직선
- 정보 행: 라벨(컬러 bold) + 값(monospace) flex row
- Tech Stack: 작은 태그 flex wrap
- SVG 아이콘에 **fill 색상 적용** (outline만이 아닌 채워진 아이콘)

**SVG 규칙:**
- viewBox는 콘텐츠에 딱 맞게 설정 (너무 크면 콘텐츠가 작아짐)
- `width:100%` + `overflow:visible` 사용
- 텍스트는 간결하게 (약어 사용 OK)
- 상세 환경 변수, 개별 볼륨은 생략하고 핵심 구조만 표현
- 나머지 색상, 아이콘은 detail 모드와 동일

---

## 검증 (Playwright Visual QA)

HTML 생성 후 반드시 Playwright MCP로 시각 검증을 수행한다:

### Step 1: 전체 페이지 스크린샷

1. 로컬 HTTP 서버 실행: `python3 -m http.server 8888` (docs 디렉토리에서)
2. Playwright로 페이지 열기: `browser_navigate` → `http://localhost:8888/{파일명}`
3. 전체 페이지 스크린샷 캡쳐: `browser_take_screenshot` (fullPage: true)

### Step 2: 패널별 상세 스크린샷 (CRITICAL)

**전체 페이지 스크린샷만으로는 디테일 문제를 발견할 수 없다.** 반드시 각 패널을 개별로 캡쳐하여 확대 검증한다:

```
browser_take_screenshot(element: "Panel 이름", ref: "패널ref", type: png)
```

각 패널 스크린샷에서 다음을 확인:
- **박스 내부 여백 (CRITICAL)**: 중첩 박스(box-in-box)의 상하좌우 여백이 균등해야 한다. 컨테이너 박스 안에 자식 박스를 배치할 때, 첫 번째 자식의 위 여백과 마지막 자식의 아래 여백이 동일해야 보기 편하다.
- **박스 간 최소 간격**: 직선 연결은 최소 40px, S-커브 연결은 최소 60px, S-커브 팬아웃(1→3)은 최소 70px.
- **화살표 머리(marker)**: 모든 라인 끝에 화살표 삼각형이 정상 표시되는가? (marker 누락, 잘림 확인)
- **라인 연결점**: 라인이 박스의 edge center에 연결되는가? (텍스트나 박스 내부에 연결되면 안 됨)
- **라벨-라인 겹침 방지 (CRITICAL)**: 화살표 옆 텍스트 라벨이 라인과 겹치면 안 된다. 라벨은 라인 위가 아닌 **라인 옆(offset)**에 배치한다:
    - 수직 라인: 라벨을 라인 오른쪽에 배치 (x = 라인x + 15px)
    - 수평 라인: 라벨을 라인 위쪽에 배치 (y = 라인y - 12px)
    - S-커브: 라벨을 커브 바깥쪽에 배치 (커브가 왼쪽으로 휘면 라벨은 왼쪽 위에)
    - 흰색 배경 rect는 사용하지 않는다 (텍스트 중앙 정렬 오차 문제)
- **라인이 박스 관통**: 라인이 다른 박스 위를 지나가지 않는가? (박스 아래로 우회해야 함)
- **화살표 방향**: 데이터 흐름 방향과 일치하는가? (orient="auto" + 역방향 line에서 반대 표시 빈번)
- **텍스트 오버플로우**: 텍스트가 박스 밖으로 넘치지 않는가?
- **박스 겹침**: 박스끼리 겹치지 않는가?

### Step 3: SVG 좌표 셀프 검증

Playwright 캡쳐 전에 HTML 소스에서 SVG 좌표를 프로그래밍적으로 검증한다:

1. **모든 path의 끝점이 target 박스의 edge center와 일치하는지 확인** (오차 ±2px)
2. **animateMotion path가 해당 화살표 path와 동일한 경로인지 확인**
3. **marker-end 속성이 모든 화살표 path에 있는지 확인**
4. **defs에 사용된 모든 색상의 marker가 정의되어 있는지 확인**

### Step 4: 수정 반복

5. 문제 발견 시 HTML을 수정하고 다시 캡쳐하여 확인
6. **최대 5회 반복** — 5회 이내에 해결되지 않으면 현재 상태로 완료하고 발견된 이슈를 사용자에게 보고

### Step 5: 최종 스크린샷 저장 (필수)

7. 검증 완료 후 최종 화면을 PNG 파일로 저장한다
   - `browser_take_screenshot` (fullPage: true, type: png)
   - 파일명: `{프로젝트경로}/docs/infrastructure[-simple]-screenshot.png`

---

## 주의사항

- 이모지를 절대 사용하지 않는다. 모든 시각적 요소는 SVG 아이콘으로 표현한다.
- 외부 CDN, 라이브러리, 폰트를 로드하지 않는다. 순수 HTML + CSS + inline SVG만 사용한다.
- 반응형 레이아웃은 불필요하다. 1600px 고정 너비를 기준으로 한다.
- SVG 내부에서 `<foreignObject>`는 사용하지 않는다. 순수 SVG 요소만 사용한다.
- 분석 시 실제 존재하는 파일만 읽는다. 추측으로 파일을 만들어내지 않는다.
- 설정 파일에서 확인할 수 없는 컴포넌트는 다이어그램에 포함하지 않는다.
- 서비스가 매우 많은 경우 (15개 초과) 주요 서비스만 포함하고 나머지는 그룹으로 표시한다.
