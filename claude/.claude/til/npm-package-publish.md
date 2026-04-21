# npm 패키지 배포 (npmjs.com)

> 작성일: 2026-03-30

## 개념

npm(Node Package Manager)은 JavaScript/TypeScript 패키지를 공유하는 공개 레지스트리.
패키지를 npmjs.com에 배포하면 누구나 `npm install`로 설치할 수 있다.

```
배포자: npm publish → npmjs.com (중앙 저장소)
사용자: npm install -g 패키지명 → 로컬에 설치
```

비유하면:
- **git 저장소** = 소스 코드 관리 (개발용)
- **npm registry** = 빌드된 패키지 배포 (배포용)
- PHP의 Packagist, Python의 PyPI와 같은 역할

---

## 1단계: npmjs.com 계정 생성

### 웹에서 가입

1. https://www.npmjs.com/signup 접속
2. 계정 생성 (username, email, password)
3. 이메일 인증

### CLI에서 로그인

```bash
npm login
# Username: 입력
# Password: 입력
# Email: 입력
# OTP: (2FA 설정 시)
```

로그인 확인:
```bash
npm whoami
# → 본인 username 출력되면 성공
```

---

## 2단계: package.json 설정

npm 배포에 필요한 필수/권장 필드:

```json
{
  "name": "lsw-admin-mcp",
  "version": "1.0.0",
  "description": "MCP server for LSW Admin CMS",
  "main": "dist/index.js",
  "bin": {
    "lsw-admin-mcp": "dist/index.js"
  },
  "files": [
    "dist/"
  ],
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build"
  },
  "keywords": ["mcp", "admin"],
  "license": "MIT",
  "engines": {
    "node": ">=18"
  }
}
```

### 주요 필드 설명

| 필드              | 역할                                                             |
|-------------------|------------------------------------------------------------------|
| `name`            | 패키지 이름 (npmjs.com에서 유일해야 함)                          |
| `version`         | semver 버전 (배포할 때마다 올려야 함)                            |
| `main`            | 패키지 진입점 (require/import 시 로드되는 파일)                  |
| `bin`             | CLI 명령어 등록 (`npm install -g` 시 실행 가능한 명령어)         |
| `files`           | npm에 포함할 파일/폴더 (나머지는 제외됨)                         |
| `prepublishOnly`  | `npm publish` 전에 자동 실행되는 스크립트                        |

### `files` 필드가 중요한 이유

`files`에 지정한 것만 npm 패키지에 포함된다.
`.env`, `src/` (TypeScript 원본), 테스트 파일 등은 자동으로 제외됨.

```
files: ["dist/"] 설정 시:

포함됨:     dist/index.js, dist/tools/dashboard.js, ...
제외됨:     src/, .env, .mcp.json, node_modules/, tests/, ...
항상 포함:  package.json, README.md, LICENSE (자동)
```

### `.npmignore` (대안)

`files` 대신 `.npmignore` 파일로 제외 목록을 지정할 수도 있다.
`.gitignore`와 문법이 동일하다.

```
# .npmignore
src/
tests/
.env
.mcp.json
tsconfig.json
```

> 권장: `files` 필드 사용 (화이트리스트 방식이 더 안전)

---

## 3단계: 빌드 (TypeScript → JavaScript)

npm에는 JavaScript만 배포한다. TypeScript 소스는 배포하지 않는다.

```bash
# tsconfig.json에 outDir 설정 필요
{
  "compilerOptions": {
    "outDir": "dist",
    "declaration": true,    // .d.ts 타입 정의 파일 생성 (선택)
    "sourceMap": false       // 소스맵 제외 (배포 시 불필요)
  }
}
```

```bash
npm run build     # tsc 실행 → dist/ 폴더에 .js 파일 생성
```

### bin 파일의 shebang

CLI로 실행할 파일(bin에 지정한 파일)에는 첫 줄에 shebang이 필요:

```javascript
#!/usr/bin/env node
// dist/index.js 첫 줄
```

TypeScript 소스(`src/index.ts`)에 추가해두면 빌드 시 자동 포함:

```typescript
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
// ...
```

---

## 4단계: 배포 전 확인

### 패키지 내용물 확인

```bash
npm pack --dry-run
```

실제로 패키지에 포함될 파일 목록이 출력된다.
민감 정보(`.env`, PAT, URL 등)가 포함되지 않는지 반드시 확인.

### 패키지 크기 확인

```bash
npm pack
# → lsw-admin-mcp-1.0.0.tgz 생성
ls -lh lsw-admin-mcp-1.0.0.tgz
```

생성된 `.tgz` 파일의 내용을 확인:

```bash
tar tzf lsw-admin-mcp-1.0.0.tgz
```

### 이름 중복 확인

```bash
npm search lsw-admin-mcp
# 또는
npm view lsw-admin-mcp
# → 404 나오면 사용 가능
```

---

## 5단계: 배포

```bash
npm publish
```

이게 전부. `prepublishOnly` 스크립트가 있으면 빌드 후 자동 배포된다.

### scoped 패키지 (선택)

이름 충돌을 피하려면 scope를 사용할 수 있다:

```json
{
  "name": "@myorg/lsw-admin-mcp"
}
```

scoped 패키지를 public으로 배포:

```bash
npm publish --access public
```

> scoped 패키지는 기본이 private이므로 `--access public` 필수

---

## 6단계: 사용자 설치

### 글로벌 설치 (CLI 도구)

```bash
npm install -g lsw-admin-mcp
```

### MCP 설정 (`.mcp.json`)

```json
{
  "mcpServers": {
    "lsw-admin": {
      "command": "lsw-admin-mcp",
      "env": {
        "ADMIN_KIC_STAGE_URL": "https://st-kic.lsw...",
        "ADMIN_KIC_STAGE_PAT": "pat_..."
      }
    }
  }
}
```

기존 `npx tsx /path/to/src/index.ts` → `lsw-admin-mcp` 로 간결해짐.

### 업데이트

```bash
npm update -g lsw-admin-mcp
```

---

## 버전 관리 (semver)

배포할 때마다 버전을 올려야 한다. 같은 버전으로 재배포 불가.

```bash
npm version patch    # 1.0.0 → 1.0.1 (버그 수정)
npm version minor    # 1.0.0 → 1.1.0 (기능 추가, 하위 호환)
npm version major    # 1.0.0 → 2.0.0 (Breaking change)
npm publish
```

`npm version` 명령은 자동으로:
1. `package.json`의 version 업데이트
2. git commit 생성
3. git tag 생성

### semver 의미

```
MAJOR.MINOR.PATCH
  │     │     └─ 버그 수정 (기존 기능 변경 없음)
  │     └─────── 기능 추가 (하위 호환 유지)
  └───────────── Breaking change (하위 호환 깨짐)
```

---

## 배포 취소 / 삭제

```bash
# 72시간 이내만 가능
npm unpublish lsw-admin-mcp@1.0.0

# 특정 버전을 deprecated 처리 (삭제 대신 권장)
npm deprecate lsw-admin-mcp@1.0.0 "보안 이슈로 1.0.1 이상 사용 권장"
```

---

## 전체 흐름 요약

```
[배포자]
  1. 코드 수정
  2. npm version patch/minor/major
  3. npm publish
     └→ prepublishOnly: tsc (빌드)
     └→ files: ["dist/"] 만 업로드
     └→ npmjs.com에 등록

[사용자]
  1. npm install -g lsw-admin-mcp (최초)
     npm update -g lsw-admin-mcp  (업데이트)
  2. .mcp.json에 env 설정 (URL, PAT)
  3. 사용
```

---

## 비교: 현재 방식 vs npm 배포

| 항목          | git pull 방식                       | npm 배포 방식                |
|---------------|-------------------------------------|------------------------------|
| 사용자 설치   | git clone + npm install             | npm install -g 패키지명      |
| 업데이트      | git pull                            | npm update -g 패키지명       |
| 빌드 주체     | 사용자 (tsx 런타임)                 | 배포자 (tsc 빌드 후 배포)    |
| 런타임 의존   | tsx 필요                            | Node.js만 필요               |
| 버전 관리     | git commit hash                     | semver (1.0.0, 1.1.0...)     |
| 롤백          | git checkout 특정커밋               | npm install -g 패키지@1.0.0  |
| 외부 접근     | git 저장소 접근 권한 필요           | 누구나 npm install           |
| 오프라인      | git clone 후 가능                   | npm install 후 가능          |
