# `npx skills add <repo>` 동작 방식과 저장소 포맷 조사

- **조사 일자**: 2026-09-07
- **한 줄 요약**: `skills`는 Vercel Labs가 배포하는 MIT 라이선스 CLI(v1.5.24)로, Git 저장소·로컬 경로·URL에서 `SKILL.md`(YAML frontmatter `name`+`description` 필수)를 가진 디렉터리를 찾아 프로젝트 canonical 위치 `.agents/skills/<name>/`에 복사한 뒤 77개 에이전트 디렉터리(`.claude/skills/` 등)로 symlink(또는 copy)하고, 프로젝트 루트에 `skills-lock.json`을 기록한다.

> 조사 기준 커밋: `vercel-labs/skills@1682051d48c34f5eb135e6475c1a965dce05e820` (main, 2026-09-06 push). 아래 소스 경로 인용은 모두 이 커밋 기준이며, npm 배포본 1.5.24의 README와 대조했다.

---

## 1. `skills` npm 패키지 정체

| 항목 | 값 |
|---|---|
| 패키지명 | `skills` (bin: `skills`, `add-skill`) |
| 현재 버전 | **1.5.24** (npm `time.modified` 2026-09-06T22:50:37Z) |
| 설명 | "The open agent skills ecosystem" |
| 저장소 | https://github.com/vercel-labs/skills (default branch `main`, ★30,550) |
| 라이선스 | MIT |
| npm maintainers | `rauchg`, `quuu` |
| 홈페이지 | https://github.com/vercel-labs/skills#readme |
| 관련 디렉터리 서비스 | https://skills.sh |

출처:
- `npm view skills` 출력(2026-09-07 실행). tarball: https://registry.npmjs.org/skills/-/skills-1.5.24.tgz
- `gh api repos/vercel-labs/skills` (license `MIT`, `pushed_at` 2026-09-06T22:47:54Z)
- tarball 내 `package/package.json` — `"bin": {"skills": "./bin/cli.mjs", "add-skill": "./bin/cli.mjs"}`

---

## 2. `skills add` 문법과 옵션

### 2.1 실제 `--help` 출력 (v1.5.24, `npx -y skills@1.5.24 --help`)

```
add <package>        Add a skill package (alias: a)
use <package>@<skill> Generate a prompt for using one skill without installing it
remove [skills]      Remove installed skills
list, ls             List installed skills
find [query]         Search for skills interactively
update [skills...]   Update skills to latest versions (alias: upgrade)
experimental_install Restore skills from skills-lock.json
init [name]          Initialize a skill (creates <name>/SKILL.md or ./SKILL.md)
experimental_sync    Sync skills from node_modules into agent directories

Add Options:
  -g, --global           Install skill globally (user-level) instead of project-level
  -a, --agent <agents>   Specify agents to install to (use '*' for all agents)
  -s, --skill <skills>   Specify skill names to install (use '*' for all skills)
  -l, --list             List available skills in the repository without installing
  -y, --yes              Skip confirmation prompts
  --copy                 Copy files instead of symlinking to agent directories
  --metadata <json>      Attach valid JSON to the install telemetry event
  --subagent <names>     Install to Eve subagents (use 'root' for the root agent)
  --all                  Shorthand for --skill '*' --agent '*' -y
  --full-depth           Search all subdirectories even when a root SKILL.md exists
```

- 서브커맨드별 `--help`(`add --help`, `init --help` 등)는 모두 동일한 전체 도움말을 출력한다(실행으로 확인).
- `add`의 alias: `a`, `i`, `install` — 출처: `src/cli.ts` L351-354 (`case 'i': case 'install': case 'a': case 'add'`).

### 2.2 허용되는 source 형식 (`src/source-parser.ts` `parseSource()`)

| 입력 형태 | 파싱 결과 `type` | 근거 (`src/source-parser.ts`) |
|---|---|---|
| `./path`, `../path`, `/abs`, `C:\...` | `local` | `isLocalPath()` L127-137 |
| `owner/repo` | `github` (GH_HOST가 github.com이 아니면 `git`) | L433-463 |
| `owner/repo/sub/path` | `github` + `subpath` | L453-463 |
| `owner/repo@skill-name` | `github` + `skillFilter` | L441-451 |
| `owner/repo#ref`, `owner/repo#ref@skill` | fragment `#`를 git ref로 해석 (git-like 소스만) | `parseFragmentRef()` L204-234 |
| `github:owner/repo`, `gitlab:owner/repo` | prefix shorthand | L297-314 |
| `https://github.com/owner/repo` | `github` | L374-384 |
| `https://github.com/owner/repo/tree/<branch>` | `github` + `ref` | L363-372 |
| `https://github.com/owner/repo/tree/<branch>/<path>` | `github` + `ref` + `subpath` | L351-361 |
| `https://gitlab.com/group/sub/repo[/-/tree/<ref>/<path>]` | `gitlab` (서브그룹 지원) | L386-431 |
| `git@github.com:owner/repo.git`, `ssh://…`, `https://….git` | `git` (fallback) | L475-480 |
| `raw.githubusercontent.com/…`, `github.com/o/r/archive/…`, `releases/download/…`, `gitlab …/-/archive/…` | `download` (직접 다운로드: 단일 `SKILL.md` 또는 `.zip/.tar/.tar.gz/.tgz`) | `isHostedArtifactUrl()` L243-270 |
| 그 외 `http(s)://` (GitHub/GitLab 아님, `.git` 미포함) | `well-known` (well-known discovery 후 download fallback) | `isWellKnownUrl()` L488-511 |
| `vercel-labs/vercel-skills`, `coinbase/agentWallet` | alias → 실제 repo로 치환 | `SOURCE_ALIASES` L145-148 |

- `subpath`는 `..` 세그먼트를 포함하면 에러 (`sanitizeSubpath()` L106-122).
- 클론은 `git clone --depth 1 [--branch <ref>]`; ref가 커밋 SHA이면 `--branch` 실패 후 `fetch --depth 1 origin <sha>` + `checkout FETCH_HEAD`로 폴백 — `src/git.ts` L295-320, L75-76.
- 다운로드 제한: 10 MiB / 압축 해제 25 MiB / 1000 파일 (`SKILLS_DOWNLOAD_MAX_BYTES`, `SKILLS_EXTRACT_MAX_BYTES`, `SKILLS_EXTRACT_MAX_FILES`로 조정) — README "Examples" 절.
- 프라이빗 저장소: Git credential helper → `gh repo clone` → SSH 순 폴백. `GITHUB_TOKEN`/`GH_TOKEN`은 선택 — README "Private Repositories" 절.

### 2.3 설치 방식(symlink vs copy)과 canonical 디렉터리

- 프로젝트 스코프 canonical 위치: `<cwd>/.agents/skills/<skill-name>` , 글로벌은 `~/.agents/skills/<skill-name>` — `src/installer.ts` `getCanonicalSkillsDir()` L98-101, `src/constants.ts` (`AGENTS_DIR='.agents'`, `SKILLS_SUBDIR='skills'`).
- 기본 모드 `symlink`: canonical에 복사 후 각 에이전트 디렉터리에 상대경로 symlink(Windows는 junction). `--copy`면 canonical을 건너뛰고 에이전트 디렉터리에 직접 복사 — `src/installer.ts` `installSkillForAgent()` L265-370, `createSymlink()` L197-258.
- 설치 디렉터리명은 `sanitizeName()`으로 소문자 kebab-case 정규화 (`[^a-z0-9._]+` → `-`, 앞뒤 `.`/`-` 제거, 255자 제한) — `src/installer.ts` L50-68.
- 복사 시 제외: 파일 `metadata.json`, 디렉터리 `.git`, `__pycache__`, `__pypackages__`; symlink는 dereference하여 실체 복사, 깨진 symlink는 건너뜀 — `src/installer.ts` L423-430, L462-505.

---

## 3. 기대하는 저장소 포맷

### 3.1 스킬 탐색 위치 (`src/skills.ts` `discoverSkills()` L178-327)

우선순위 순서:

1. **`searchPath` 루트에 `SKILL.md`가 있으면** 그 하나만 반환하고 종료 (`--full-depth` 시 계속 탐색). 단, 그 루트 스킬이 `skills-lock.json`에 기록된 "설치된 프로젝트 스킬"이면 무시하고 계속 — L232-248.
2. `prioritySearchDirs` 순회 (L251-258):
   - `<root>` (depth 1만)
   - `skills/`, `skills/.curated/`, `skills/.experimental/`, `skills/.system/` (depth 3까지)
   - `AGENT_PROJECT_SKILL_DIRS` (depth 3까지, L12-41): `.agents/skills`, `.claude/skills`, `.cline/skills`, `.codebuddy/skills`, `.codex/skills`, `.commandcode/skills`, `.continue/skills`, `.github/skills`, `.goose/skills`, `.grok/skills`, `.iflow/skills`, `.junie/skills`, `.kilocode/skills`, `.kimchi/skills`, `.kiro/skills`, `.minimax/skills`, `.mux/skills`, `.neovate/skills`, `.opencode/skills`, `.openhands/skills`, `.pi/skills`, `.posit/assistant/skills`, `.qoder/skills`, `.roo/skills`, `.trae/skills`, `.windsurf/skills`, `.zcode/skills`, `.zencoder/skills`
   - 플러그인 매니페스트가 선언한 경로 (depth 1) — `getPluginSkillPaths()`
3. 컨테이너 디렉터리 walk 깊이: `DEFAULT_SKILL_CONTAINER_DEPTH = 3` (`src/constants.ts`) → `skills/<name>/`, `skills/<cat>/<name>/`, `skills/<cat>/<cat>/<name>/` 모두 발견. 상위에서 `SKILL.md`를 찾으면 그 아래로는 내려가지 않음(shadowing) — `walkSkillDirs()` L282-301.
4. 아무것도 못 찾았거나 `--full-depth`이면 **재귀 전체 탐색**(최대 depth 5, `node_modules`/`.git`/`dist`/`build`/`__pycache__` 제외) — `findSkillDirs()` L133-155, L308-324.
5. 이름 중복 시 먼저 발견된 것만 채택 (`seenNames`).

### 3.2 `SKILL.md` frontmatter 파싱 규칙 (CLI 측)

- 파서: `src/frontmatter.ts` — 정규식 `^---\r?\n([\s\S]*?)\r?\n---` 로 YAML 블록만 추출, `yaml` 패키지로 파싱. `---js` 등 코드 frontmatter는 의도적으로 미지원.
- 필수: `name`, `description` — 둘 다 존재해야 하고 **string 타입**이어야 함. 아니면 경고 후 스킵 — `src/skills.ts` `parseSkillMd()` L98-113.
- `metadata.internal: true` → 기본 숨김. `INSTALL_INTERNAL_SKILLS=1|true` 또는 `--skill`로 명시 지정 시에만 설치 — L115-122, `shouldInstallInternalSkills()` L59-62.
- CLI는 `name`의 길이/문자 제약을 **검증하지 않는다**(spec의 64자·소문자 규칙은 CLI 코드에 없음). 대신 설치 디렉터리명을 `sanitizeName()`으로 정규화한다 — `src/installer.ts` L50-68.
- `license`, `compatibility`, `allowed-tools` 등 다른 필드는 CLI가 그대로 통과시킴(파싱만 하고 검증 없음). 예외: Eve 에이전트 대상 설치 시 `description`/`license` 등 일부만 남기고 frontmatter를 재작성 — `stripIgnoredEveFrontmatter()` `src/installer.ts` L432-460.

### 3.3 매니페스트 / 락 파일

| 파일 | 위치 | 역할 | 출처 |
|---|---|---|---|
| `skills-lock.json` | 프로젝트 루트 (`<cwd>/skills-lock.json`) | 프로젝트 스코프 설치 기록. `{version:1, skills:{<name>:{source, sourceUrl?, ref?, sourceType, skillPath?, computedHash, subagents?, wellKnownDigest?}}}`. 타임스탬프 없음(merge conflict 최소화), 이름순 정렬. **VCS에 커밋하도록 설계**. | `src/local-lock.ts` L5-66; 기록 시점 `src/add.ts` L937-960 (`!installGlobally`일 때만), L1900-1930 |
| `~/.agents/.skill-lock.json` (또는 `$XDG_STATE_HOME/skills/.skill-lock.json`) | 사용자 홈 | 글로벌 락 v3. `skillFolderHash`(GitHub tree SHA), `installedAt/updatedAt`, `lastSelectedAgents` 등 | `src/skill-lock.ts` L6-76 |
| `.claude-plugin/marketplace.json`, `.claude-plugin/plugin.json` | 저장소 루트 | Claude Code 플러그인 매니페스트. `plugins[].skills: ["./skills/x"]` 경로(반드시 `./`로 시작)와 `metadata.pluginRoot`를 읽어 탐색 경로에 추가 | `src/plugin-manifest.ts` L15-80; README "Plugin Manifest Discovery" |
| `metadata.json` | 스킬 디렉터리 내 | 설치 시 **복사에서 제외**되는 파일 | `src/installer.ts` L423 |

- `skills.json`, `.skillsrc` 같은 저장소 측 매니페스트는 **소스에 존재하지 않음** (`src/` 전체 grep 결과 없음). 저장소 작성자가 만들어야 할 매니페스트는 없고, `SKILL.md`만 있으면 된다.
- `npx skills experimental_install`은 `skills-lock.json`을 읽어 `.agents/skills/`(universal)로만 복원한다 — `src/install.ts` L10-18.

### 3.4 지원 파일(scripts/, references/, assets/) 처리

- CLI는 스킬 디렉터리 **전체를 재귀 복사**한다(제외 목록 제외). 하위 디렉터리 이름에 대한 특별 취급 없음 — `src/installer.ts` `copyDirectory()` L462-505. 파일 모드(`chmod`)는 원본 보존 (L495-496) → `scripts/*.sh` 실행 권한 유지.
- `scripts/`, `references/`, `assets/` 관례는 Agent Skills spec에서 오는 것이며(§5 참조), CLI가 강제하지 않는다.

---

## 4. 설치 대상 에이전트와 경로

- 에이전트 정의: `src/agents.ts` `agents` 객체 (각 항목 `skillsDir`, `globalSkillsDir`, `detectInstalled`), 타입 목록 `src/types.ts` `AgentType` (77개 + `universal`).
- README의 "Supported Agents" 표는 `scripts/sync-agents.ts`가 `agents.ts`에서 자동 생성 (`<!-- supported-agents:start -->` 마커).
- 설치된 에이전트는 `detectInstalled()`(예: `~/.claude` 존재 여부)로 자동 감지; 없으면 선택 프롬프트 — README "Supported Agents" 하단 NOTE.

주요 에이전트 발췌 (전체 표는 README 참조):

| Agent | `--agent` | Project Path | Global Path | 소스 |
|---|---|---|---|---|
| Claude Code | `claude-code` | `.claude/skills/` | `~/.claude/skills/` | `src/agents.ts` L152-160 |
| Codex | `codex` | `.agents/skills/` | `~/.codex/skills/` | L219-227 |
| Cursor | `cursor` | `.agents/skills/` | `~/.cursor/skills/` | L264-272 |
| Gemini CLI | `gemini-cli` | `.agents/skills/` | `~/.gemini/skills/` | L341-349 |
| GitHub Copilot | `github-copilot` | `.agents/skills/` | `~/.copilot/skills/` | L350-358 |
| OpenCode | `opencode` | `.agents/skills/` | `~/.config/opencode/skills/` | README 표 |
| Windsurf | `windsurf` | `.windsurf/skills/` | `~/.codeium/windsurf/skills/` | README 표 |
| Amp / Replit / Universal | `amp`,`replit`,`universal` | `.agents/skills/` | `~/.config/agents/skills/` | README 표 |
| Cline, Dexto, Kimi Code CLI, Loaf, Warp, Zed | (각 id) | `.agents/skills/` | `~/.agents/skills/` | README 표 |
| Eve | `eve` | `agent/skills/` | N/A (project-only) | README 표 |

- 프로젝트 경로가 `.agents/skills/`인 에이전트("universal")는 canonical 위치와 동일하므로 symlink를 만들지 않는다 — `src/installer.ts` L115-121 주석, `src/agents.ts` L839-880 (`getUniversalAgents()`, `isUniversalAgent()`).

---

## 5. Agent Skills 스펙(agentskills.io)과의 관계

- README는 "Skills are generally compatible across agents since they follow a shared [Agent Skills specification](https://agentskills.io)"라고 명시하고, Related Links에 spec을 첫 번째로 둔다 — README "Compatibility", "Related Links" 절.
- 스펙 출처: **Anthropic이 처음 개발해 오픈 표준으로 공개**, GitHub `agentskills/agentskills`에서 관리 — https://agentskills.io/ "Open development" 절.
- 스펙 `SKILL.md` frontmatter (https://agentskills.io/specification):

| 필드 | 필수 | 제약 |
|---|---|---|
| `name` | 예 | 1–64자, 소문자 영숫자(`a-z`,`0-9`)와 `-`만, 하이픈으로 시작/끝 불가, `--` 연속 불가, **부모 디렉터리명과 일치해야 함** |
| `description` | 예 | 1–1024자, 비어 있으면 안 됨. 무엇을 하는지 + 언제 쓰는지 |
| `license` | 아니오 | 라이선스명 또는 번들 라이선스 파일 참조 |
| `compatibility` | 아니오 | 1–500자. 환경 요구사항 |
| `metadata` | 아니오 | string→string 맵 |
| `allowed-tools` | 아니오 | 공백 구분 문자열 (Experimental) |

- 디렉터리 구조: `SKILL.md`(필수) + `scripts/`, `references/`, `assets/`(선택, 관례). `SKILL.md` 본문 500줄 이하 권장, 파일 참조는 스킬 루트 기준 상대경로 1단계 권장 — 같은 페이지 "Directory structure", "Progressive disclosure", "File references".
- 검증 도구: `skills-ref validate ./my-skill` (https://github.com/agentskills/agentskills/tree/main/skills-ref) — 같은 페이지 "Validation".
- Claude Code 측 규칙 (https://code.claude.com/docs/en/skills):
  - Claude Code는 spec을 "extensions와 함께 구현". claude.ai 업로드/Skills API에서 허용되는 필드는 `name, description, license, compatibility, metadata, allowed-tools`뿐이며, `disable-model-invocation, user-invocable, argument-hint, arguments, disallowed-tools, model, effort, context, agent, background, hooks, paths, shell`은 Claude Code 전용 확장.
  - Claude Code 로컬에서는 `name`이 선택(디렉터리명 기본값), `description`+`when_to_use` 합산 **1,536자**에서 잘림.
  - 로드 위치: `~/.claude/skills/<name>/SKILL.md`(personal), `.claude/skills/<name>/SKILL.md`(project), `<plugin>/skills/<name>/SKILL.md`.
- CLI README의 호환성 표: `allowed-tools`는 대부분 에이전트 지원(Kiro CLI, Zencoder 제외), `context: fork`와 Hooks는 Claude Code(Hooks는 Cline, Kiro CLI도) 한정 — README "Compatibility" 표.

**정리**: `skills` CLI가 요구하는 최소 조건(`name`/`description` string)은 spec의 부분집합이고, spec의 이름 규칙·길이 제한은 CLI가 검증하지 않는다. 따라서 저장소 작성자는 spec(64자 소문자 kebab-case, 디렉터리명 일치, description ≤1024자)을 따르면 CLI와 모든 에이전트에서 안전하다.

---

## 6. 저장소 작성자에게 유의미한 기타 명령

| 명령 | 동작 | 출처 |
|---|---|---|
| `npx skills init [name]` | `<name>/SKILL.md`(또는 `./SKILL.md`, 이름은 cwd basename) 템플릿 생성. 이미 있으면 스킵. 다음 단계로 `npx skills add <owner>/<repo>` 안내 출력 | `src/cli.ts` `runInit()` L231-290 |
| `npx skills add <src> --list` / `-l` | 설치 없이 저장소 내 발견되는 스킬 목록 출력 → 레이아웃 검증에 유용 | `--help`, README |
| `npx skills add <src> --full-depth` | 루트 `SKILL.md`가 있어도 하위 전체 탐색 | `--help`; `src/skills.ts` L243-246, L309 |
| `npx skills update [skills...]` / `upgrade` / **`check`** | 설치된 스킬을 최신으로 갱신. `-g`/`-p`/`-y`. `check`는 문서화되지 않은 alias로 동일 함수(`runUpdate`) 호출 | `src/cli.ts` L391-395 |
| `npx skills find [query] [--owner <owner>]` | skills.sh 기반 검색(인터랙티브/키워드) | `--help`, README |
| `npx skills list` / `ls [-g] [-a …] [--json]` | 설치된 스킬 목록 | `--help` |
| `npx skills use <src>@<skill> [--agent]` | 설치 없이 프롬프트 생성/에이전트 실행 | README "Use a Skill Without Installing" |
| `npx skills experimental_install` | `skills-lock.json`에서 `.agents/skills/`로 복원 | `src/install.ts` |
| `npx skills experimental_sync` | `node_modules` 내 스킬을 에이전트 디렉터리로 동기화 | `--help` |
| `npx skills remove` / `rm` | 제거 (`--all`, `--skill '*'`, `--agent '*'`) | README |

환경변수: `INSTALL_INTERNAL_SKILLS`, `DISABLE_TELEMETRY`, `DO_NOT_TRACK`, `GITHUB_TOKEN`, `GH_TOKEN`, `GH_HOST`(GitHub Enterprise; `src/source-parser.ts` L435-439), `XDG_STATE_HOME` — README "Environment Variables", 소스.

---

## 7. 최소 예시 저장소 레이아웃

```
my-skills-repo/
├── README.md
├── skills/                        # CLI 우선 탐색 컨테이너 (depth 3까지)
│   ├── pr-review/
│   │   ├── SKILL.md               # 필수. name은 디렉터리명과 동일하게
│   │   ├── scripts/               # 선택 (spec 관례) — 실행 권한 보존됨
│   │   │   └── collect-diff.sh
│   │   ├── references/            # 선택
│   │   │   └── CHECKLIST.md
│   │   └── assets/                # 선택
│   │       └── template.md
│   └── release-notes/
│       └── SKILL.md
└── skills-lock.json               # (선택) 이 저장소가 다른 스킬을 '설치'한 경우에만 생성됨
```

- 대안 레이아웃도 동일하게 동작: 루트 단일 스킬(`./SKILL.md`), `.claude/skills/<name>/SKILL.md`, `.agents/skills/<name>/SKILL.md`, `skills/<category>/<name>/SKILL.md` — `src/skills.ts` L251-258, L260-266.
- Claude Code 플러그인 마켓플레이스 호환이 필요하면 `.claude-plugin/marketplace.json`에 `"skills": ["./skills/pr-review"]` 형식(경로는 `./`로 시작) — `src/plugin-manifest.ts` L15-19, README "Plugin Manifest Discovery".

설치 명령 예시:

```bash
npx skills add owner/my-skills-repo --list                       # 레이아웃 검증
npx skills add owner/my-skills-repo --skill pr-review -a claude-code -y
npx skills add owner/my-skills-repo/skills/pr-review              # subpath 직접 지정
npx skills add owner/my-skills-repo@pr-review                     # @skill 필터
npx skills add owner/my-skills-repo#v1.2.0                        # 태그/브랜치 ref
```

## 8. 최소 `SKILL.md` 예시

```markdown
---
name: pr-review
description: Review pull requests against the team checklist. Use when the user asks to review a PR, check a diff, or prepare a code review.
license: MIT
metadata:
  author: my-org
  version: "1.0"
---

# PR Review

## When to use
User asks to review a PR or diff.

## Steps
1. Run `scripts/collect-diff.sh` to gather the diff.
2. Check each item in [references/CHECKLIST.md](references/CHECKLIST.md).
3. Report findings grouped by severity.
```

- `name`/`description`은 CLI 필수(`src/skills.ts` L98-104), 나머지는 spec 선택 필드(https://agentskills.io/specification). `skills init`이 생성하는 템플릿은 `name`, `description`만 포함 (`src/cli.ts` L250-266).

---

## 9. 확인 못 한 것 / 불확실한 것

1. **README 탐색 경로 목록 vs 실제 코드 불일치**: README `<!-- skill-discovery -->` 목록(예: `.aider-desk/skills/`, `.augment/skills/`, `.bob/skills/` 등 58개)은 `scripts/sync-agents.ts`가 `agents.ts`의 `skillsDir`에서 생성하지만, 실제 우선 탐색은 `src/skills.ts`의 하드코딩된 `AGENT_PROJECT_SKILL_DIRS`(28개, `.github/skills`·`.codex/skills` 등 README에 없는 항목 포함)를 쓴다. README에만 있는 경로는 "우선 탐색"이 아니라 재귀 폴백(3.1-4단계)으로만 발견된다. 이 해석은 코드 읽기에 근거하며 실제 실행으로 각 경로를 검증하지는 않았다.
2. **`skills check`**: `src/cli.ts`에서 `update`의 alias임은 확인했으나, `--help`/README에 문서화되어 있지 않아 향후 제거될 수 있다.
3. **에이전트 경로 표의 전체 77개 항목**은 README(자동 생성)로만 확인했고, `src/agents.ts`에서 직접 대조한 항목은 Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot 5개뿐이다.
4. **well-known 프로토콜의 정확한 엔드포인트 규격**(`src/providers/wellknown.ts`)과 `download` 흐름의 세부(`src/download-source.ts`, `src/archive.ts`)는 이 조사 범위에서 소스를 읽지 않았다.
5. **spec의 `name`이 "unicode lowercase alphanumeric"이라고 쓰인 부분**: 표에는 `a-z`, `0-9`로 되어 있어 유니코드 범위가 모호하다. `skills-ref` 검증기 구현은 확인하지 않았다.
6. Claude Code 문서의 세부 수치(1,536자 cap, `v2.1.218+` 등)는 2026-09-07 시점 페이지 내용이며 버전에 따라 바뀔 수 있다.
7. `npm view skills`의 `author` 필드는 비어 있어 "Vercel" 명의는 GitHub org(`vercel-labs`)와 README 링크로만 뒷받침된다.

---

## 10. 출처 목록

**npm / 패키지**
- `npm view skills` (2026-09-07) — https://www.npmjs.com/package/skills
- tarball https://registry.npmjs.org/skills/-/skills-1.5.24.tgz → `package/README.md`, `package/package.json`
- `npx -y skills@1.5.24 --help` 실행 출력 (2026-09-07)

**GitHub 소스 (vercel-labs/skills @ `1682051d48c34f5eb135e6475c1a965dce05e820`)**
- https://github.com/vercel-labs/skills
- `src/cli.ts` — https://raw.githubusercontent.com/vercel-labs/skills/1682051d48c34f5eb135e6475c1a965dce05e820/src/cli.ts
- `src/skills.ts` — …/src/skills.ts (`discoverSkills`, `parseSkillMd`, `findSkillDirs`, `filterSkills`)
- `src/frontmatter.ts` — …/src/frontmatter.ts (`parseFrontmatter`)
- `src/source-parser.ts` — …/src/source-parser.ts (`parseSource`, `parseFragmentRef`, `isHostedArtifactUrl`, `isWellKnownUrl`, `sanitizeSubpath`)
- `src/installer.ts` — …/src/installer.ts (`installSkillForAgent`, `getCanonicalSkillsDir`, `sanitizeName`, `copyDirectory`, `createSymlink`)
- `src/agents.ts` — …/src/agents.ts (`agents`, `getUniversalAgents`)
- `src/types.ts` — …/src/types.ts (`AgentType`, `Skill`, `ParsedSource`)
- `src/constants.ts` — …/src/constants.ts
- `src/local-lock.ts` — …/src/local-lock.ts (`skills-lock.json`)
- `src/skill-lock.ts` — …/src/skill-lock.ts (`~/.agents/.skill-lock.json`)
- `src/plugin-manifest.ts` — …/src/plugin-manifest.ts (`getPluginSkillPaths`)
- `src/install.ts` — …/src/install.ts (`runInstallFromLock`)
- `src/git.ts` — …/src/git.ts (`cloneRepo`)
- `src/add.ts` — …/src/add.ts (락 기록 시점)
- `scripts/sync-agents.ts` — README 표 생성 스크립트

**스펙 / 문서**
- Agent Skills Specification — https://agentskills.io/specification
- Agent Skills Overview ("Open development") — https://agentskills.io/
- agentskills 저장소 및 `skills-ref` — https://github.com/agentskills/agentskills
- Claude Code Skills 문서 — https://code.claude.com/docs/en/skills
- skills.sh 디렉터리 — https://skills.sh
