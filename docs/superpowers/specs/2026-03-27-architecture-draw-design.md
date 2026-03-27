# architecture-draw Skill Design

## Overview

소스 코드를 자동 분석하여 ByteByteDgo 스타일의 아키텍처 다이어그램을 생성하는 Claude Code skill.

- **Skill 이름**: `architecture-draw`
- **실행**: `/architecture-draw` 또는 `/architecture-draw /path/to/project`
- **출력**: `docs/architecture.html` (단일 파일, 외부 의존성 없음)
- **결과물 구성**: 시스템 아키텍처 개요 + 애니메이션 데이터 흐름

## 분석 단계

Skill이 Claude에게 지시하는 코드베이스 분석 절차:

1. **프로젝트 타입 감지** — 모노레포, 마이크로서비스, 모놀리스 등 구조 파악
2. **서비스/모듈 식별** — 각 서비스의 역할, 포트, 기술 스택
3. **인프라 컴포넌트 감지** — DB, 캐시, 메시지 큐, 서비스 디스커버리 등 (설정 파일, docker-compose, 환경변수 등에서 추출)
4. **API 엔드포인트 추출** — 컨트롤러/라우터에서 HTTP 엔드포인트 파싱
5. **서비스 간 통신 추적** — HTTP 클라이언트 호출, 메시징 패턴(publish/subscribe), 이벤트 등
6. **데이터 흐름 매핑** — 위 정보를 종합해서 주요 흐름 경로를 구성

분석 대상 파일 패턴:
- 설정: `**/application*.yml`, `**/application*.properties`, `**/*.env`, `**/docker-compose*.yml`
- 빌드: `**/build.gradle*`, `**/pom.xml`, `**/package.json`, `**/go.mod`, `**/Cargo.toml`
- 라우팅: `**/*Controller*`, `**/*Router*`, `**/*Route*`, `**/routes/**`, `**/api/**`
- 통신: `**/*Client*`, `**/*Feign*`, `**/*Producer*`, `**/*Consumer*`, `**/*Listener*`
- README: `**/README.md`

## 데이터 모델

분석 결과를 정리하는 내부 구조:

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
      "type": "database | cache | messageQueue | discovery | monitoring",
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
      "name": "인증 흐름",
      "color": "#4A90D9",
      "path": ["Client", "api-gateway", "auth-service", "MySQL"]
    }
  ]
}
```

## HTML 생성 규칙

### ByteByteDgo 스타일 가이드

**레이아웃**:
- 상단: 시스템 아키텍처 전체 조감도
- 하단: 데이터 흐름 애니메이션 섹션

**색상 팔레트 (컴포넌트 유형별)**:
- 서비스: `#4A90D9` (파랑)
- 게이트웨이/프록시: `#2C3E6B` (남색)
- DB/스토리지: `#27AE60` (초록)
- 캐시: `#F39C12` (주황)
- 메시지 큐: `#8E44AD` (보라)
- 클라이언트/프론트엔드: `#7F8C8D` (회색)
- 모니터링: `#E74C3C` (빨강)

**박스 스타일**:
- 둥근 모서리 (`border-radius: 12px`)
- 단색 채우기 + 흰색 텍스트
- 컴포넌트 유형별 인라인 SVG 아이콘
- 그림자 효과 (`box-shadow`)

**연결선**:
- SVG `<line>` 또는 `<path>` 로 화살표 표현
- `marker-end`로 화살표 머리

**애니메이션 (데이터 흐름)**:
- CSS `@keyframes`로 작은 원(dot)이 경로를 따라 이동
- `<animateMotion>` SVG 애니메이션 또는 CSS `offset-path`
- 흐름별 색상 구분
- 무한 반복 (`animation: infinite`)
- 흐름 이름 라벨 표시

**기술 제약**:
- 외부 CDN/라이브러리 의존성 없음
- 순수 HTML + CSS + inline SVG
- 단일 파일로 완결 (오프라인에서 열림)
- 반응형 불필요 (고정 너비 1200px 기준)

## Skill 프롬프트 구조

```
frontmatter:
  name: architecture-draw
  description: 소스 코드를 분석하여 ByteByteDgo 스타일 아키텍처 다이어그램 HTML을 생성

본문:
  1. 분석 절차 (단계별 지시)
  2. 데이터 모델 (분석 결과 정리 구조)
  3. ByteByteDgo 스타일 가이드 (색상, 박스, 아이콘, 레이아웃)
  4. SVG 아이콘 정의 (서버, DB, 캐시, 큐, 브라우저 등)
  5. HTML 생성 규칙 (단일 파일, 애니메이션)
  6. 출력 (docs/architecture.html에 Write)
```

## 실행 흐름

```
/architecture-draw [경로]
    -> 대상 프로젝트 경로 결정 (인자 없으면 현재 디렉토리)
    -> 코드베이스 분석 (Glob, Grep, Read 사용)
    -> 분석 결과를 내부 데이터 모델로 정리
    -> HTML 생성 + Write로 파일 저장
    -> 완료 메시지 출력
```
