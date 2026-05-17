# AI 주도 개발 환경 구축 가이드 (Rocky Linux 9.4)

이 문서는 순정 상태의 Rocky Linux 9.4 서버에 AI 에이전트(Claude Code, Cursor)와 외부 컨텍스트(Notion, Stitch)를 연동하기 위한 환경 셋업 가이드입니다.

## 1. 시스템 업데이트 및 필수 빌드 도구 설치
가장 먼저 패키지 매니저를 업데이트하고, 향후 패키지 빌드에 필요한 C++ 컴파일러와 Make 등을 설치합니다.

```bash
sudo dnf update -y
sudo dnf install -y gcc-c++ make curl git wget tar
```

## 2. Node.js 및 npm 설치 (LTS 버전)
Rocky 9 기본 리포지토리의 Node.js는 버전이 낮을 수 있으므로 NodeSource를 통해 20.x(LTS) 버전을 설치합니다.

```bash
# NodeSource 리포지토리 등록
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -

# Node.js 및 npm 설치
sudo dnf install -y nodejs

# 설치 확인
node -v
npm -v
```

## 3. Python 환경 구성 (선택이지만 권장)
일부 MCP 서버나 자동화 스크립트에서 Python을 요구할 수 있으므로, 기본 Python3과 pip를 세팅합니다.

```bash
sudo dnf install -y python3 python3-pip
python3 -m pip install --upgrade pip
```

## 4. 전역 AI CLI 도구 설치
터미널 환경에서 코딩과 명령을 수행할 Claude Code를 전역으로 설치합니다.

```bash
# Claude Code 설치
sudo npm install -g @anthropic-ai/claude-code

# Anthropic API 키 인증 (실행 후 브라우저 또는 토큰으로 로그인)
claude auth
```

## 5. 프로젝트 초기화 및 디렉토리 스캐폴딩
개발을 진행할 워크스페이스를 만들고 컨텍스트 동기화를 위한 폴더 구조를 잡습니다.

```bash
mkdir -p ~/workspace/my-ai-project
cd ~/workspace/my-ai-project

# npm 프로젝트 초기화
npm init -y

# 컨텍스트 저장을 위한 docs 폴더 생성
mkdir -p docs/prd docs/design
```

## 6. Notion ➡️ MD 자동화 스크립트용 의존성 설치
Notion API를 호출해 기획서를 마크다운으로 변환하는 데 필요한 라이브러리를 설치합니다.

```bash
npm install @notionhq/client notion-to-md dotenv
```

## 7. 환경 변수 및 AI 컨텍스트 파일 세팅

### 7.1 `.env` 파일 생성
프로젝트 루트에 `.env` 파일을 생성하고 발급받은 API 키들을 입력합니다. (해당 파일은 `.gitignore`에 반드시 추가)

```env
# .env
NOTION_API_KEY="secret_..."
NOTION_DATABASE_ID="your_database_id"
STITCH_API_KEY="your_stitch_key"
```

### 7.2 `.cursorrules` (Cursor IDE용 지시어)
프로젝트 루트에 `.cursorrules` 파일을 생성하여 Cursor 에이전트가 항상 문서를 참조하도록 강제합니다.

```markdown
# .cursorrules
- 프로젝트 기획 및 요구사항은 항상 `docs/prd/PRD.md`를 우선으로 확인하라.
- UI 컴포넌트를 작성할 때는 반드시 `docs/design/DESIGN.md`의 디자인 토큰과 가이드를 참조하라.
- 임의로 색상이나 여백을 하드코딩하지 말고, 스크립트가 제공한 컨텍스트를 유지하라.
```

### 7.3 `CLAUDE.md` (Claude Code용 지시어)
터미널에서 작동하는 Claude Code를 위한 룰셋을 정의합니다.

```markdown
# CLAUDE.md
## Build & Sync Commands
- Notion 동기화: `node scripts/sync-notion.js`
- 디자인 토큰 업데이트: `mcp 호출을 통해 Stitch 디자인 최신화`

## Code Style
- React/TypeScript 환경을 가정하며, 컴포넌트는 함수형으로 작성한다.
- 디자인 적용 전 항상 docs를 먼저 읽는다.
```

## 8. Stitch MCP 연결 세팅
Stitch의 디자인 컨텍스트를 가져오기 위해 MCP 서버를 설정합니다. (Cursor의 경우 GUI의 MCP 설정 메뉴에서 추가)

**Claude Code용 설정 (`~/.claude.json` 또는 전역 설정에 추가):**
```json
{
  "mcpServers": {
    "stitch": {
      "command": "npx",
      "args": ["-y", "@google/stitch-mcp"],
      "env": {
        "STITCH_API_KEY": "YOUR_API_KEY"
      }
    }
  }
}