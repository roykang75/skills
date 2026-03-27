# Claude Code Skills Collection

Claude Code에서 사용하는 커스텀 스킬 모음입니다.

## 스킬 목록

| 스킬 | 설명 |
|---|---|
| [architecture-draw](./architecture-draw/) | 소스 코드를 분석하여 ByteByteDgo 스타일 아키텍처 다이어그램 HTML 생성 |

## 설치

개별 스킬을 심볼릭 링크로 등록합니다.

```bash
# 저장소 클론
git clone https://github.com/roykang75/skills.git ~/Workspace/skills

# 원하는 스킬을 심볼릭 링크로 등록
ln -sf ~/Workspace/skills/architecture-draw ~/.claude/skills/architecture-draw
```

## 업데이트

```bash
cd ~/Workspace/skills
git pull
```

심볼릭 링크로 설치한 경우 `git pull`만으로 최신 버전이 적용됩니다.

---

## Claude Code 스킬 관리 가이드

Claude Code에서 스킬을 추가, 삭제, 업데이트하는 일반적인 방법을 정리합니다.

> 참고: [anthropics/skills](https://github.com/anthropics/skills), [Agent Skills Specification](https://agentskills.io/specification), [Claude Code Skills 문서](https://code.claude.com/docs/en/skills)

### 스킬이란?

스킬은 `SKILL.md` 파일이 포함된 디렉토리입니다. Claude Code가 특정 작업을 수행할 때 참조하는 지시사항(프롬프트)을 정의합니다.

```
my-skill/
├── SKILL.md          # 필수: 메타데이터 + 지시사항
├── scripts/          # 선택: 실행 스크립트
├── references/       # 선택: 참고 문서
└── assets/           # 선택: 템플릿, 정적 리소스
```

### SKILL.md 형식

YAML 프론트매터 + 마크다운 본문으로 구성됩니다.

```yaml
---
name: my-skill
description: 스킬 설명. Use when the user asks about X.
---

# My Skill

Claude가 따를 지시사항을 여기에 작성한다.
```

주요 프론트매터 필드:

| 필드 | 필수 | 설명 |
|---|---|---|
| `name` | 권장 | 스킬 이름 (소문자, 하이픈만 허용, 최대 64자) |
| `description` | 권장 | 스킬 설명 (자동 호출 판단에 사용됨) |
| `allowed-tools` | 선택 | 사전 승인할 도구 목록 (예: `Read Grep Bash(git:*)`) |
| `disable-model-invocation` | 선택 | `true`면 자동 호출 비활성화, `/name`으로만 실행 |
| `user-invocable` | 선택 | `false`면 `/` 메뉴에서 숨김 |
| `context` | 선택 | `fork`면 격리된 서브에이전트에서 실행 |
| `agent` | 선택 | `context: fork` 시 서브에이전트 타입 |
| `model` | 선택 | 스킬 실행 시 모델 오버라이드 |
| `paths` | 선택 | 자동 활성화 조건 (glob 패턴) |

본문에서 사용 가능한 변수:

| 변수 | 설명 |
|---|---|
| `$ARGUMENTS` | `/skill-name` 뒤에 전달된 모든 인자 |
| `$ARGUMENTS[N]` 또는 `$N` | N번째 인자 (0-based) |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID |
| `${CLAUDE_SKILL_DIR}` | 스킬 디렉토리의 절대 경로 |

### 스킬 추가

**개인 스킬** (모든 프로젝트에 적용):

```bash
mkdir -p ~/.claude/skills/my-skill
# ~/.claude/skills/my-skill/SKILL.md 파일 생성
```

**프로젝트 스킬** (해당 프로젝트에만 적용, git 커밋으로 팀 공유 가능):

```bash
mkdir -p .claude/skills/my-skill
# .claude/skills/my-skill/SKILL.md 파일 생성
```

**심볼릭 링크** (외부 저장소에서 관리하는 경우):

```bash
ln -sf /path/to/skills/my-skill ~/.claude/skills/my-skill
```

**플러그인으로 설치**:

```bash
# GitHub 저장소를 마켓플레이스로 추가
/plugin marketplace add owner/repo

# 마켓플레이스에서 스킬 설치
/plugin install skill-name@marketplace-name

# 로컬 플러그인 (개발/테스트용)
claude --plugin-dir ./my-plugin
```

적용 우선순위: `enterprise > 개인 (~/.claude/skills/) > 프로젝트 (.claude/skills/)`

### 스킬 삭제

```bash
# 개인 스킬 삭제
rm -rf ~/.claude/skills/my-skill

# 프로젝트 스킬 삭제
rm -rf .claude/skills/my-skill

# 플러그인 스킬 삭제
/plugin uninstall plugin-name@marketplace-name
```

삭제 후 현재 세션에 반영: `/reload-plugins`

### 스킬 업데이트

**독립 스킬**: `SKILL.md`를 직접 편집. 새 세션에서 자동 반영되며, `/reload-plugins`로 즉시 반영 가능.

**플러그인 스킬**: `/plugin marketplace update marketplace-name`

**심볼릭 링크**: 원본 저장소에서 `git pull`하면 자동 반영.

### 스킬 실행

```bash
/my-skill                          # 직접 실행
/my-skill arg1 arg2                # 인자와 함께 실행
/plugin-name:skill-name            # 플러그인 스킬 실행
```

### `/plugin` 명령으로 스킬 관리

Claude Code 내에서 `/plugin` 명령을 사용하면 대화형 UI 또는 CLI로 스킬을 관리할 수 있습니다.

```bash
# 대화형 UI 열기 (Discover / Installed / Marketplaces 탭)
/plugin

# 마켓플레이스 추가
/plugin marketplace add owner/repo

# 마켓플레이스 업데이트 (최신 버전 동기화)
/plugin marketplace update marketplace-name

# 스킬 설치 (--scope: user / project / local)
/plugin install skill-name@marketplace-name
/plugin install skill-name@marketplace-name --scope project

# 스킬 삭제
/plugin uninstall skill-name@marketplace-name

# 변경사항 즉시 반영
/reload-plugins
```

대화형 UI(`/plugin`)에서는 Discover 탭에서 검색/설치, Installed 탭에서 삭제/활성화 전환, Marketplaces 탭에서 업데이트/자동 업데이트 설정이 가능합니다.

### 레거시 커맨드와의 호환

`.claude/commands/` 디렉토리의 마크다운 파일도 여전히 작동합니다. 같은 이름이 존재하면 스킬이 우선합니다.

### 플러그인으로 배포

스킬을 팀/커뮤니티에 배포하려면 플러그인으로 패키징합니다.

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       # 매니페스트 (name, description, version, author)
├── skills/
│   └── my-skill/
│       └── SKILL.md
├── agents/               # 선택
├── hooks/
│   └── hooks.json        # 선택
└── settings.json         # 선택
```

GitHub 저장소에 푸시하면 `/plugin marketplace add owner/repo`로 설치할 수 있습니다.

> 플러그인 시스템은 Claude Code **1.0.33** 이상에서 지원됩니다 (`claude --version`으로 확인).
