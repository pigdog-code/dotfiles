# Dotfiles

여러 머신에서 동일한 개발 환경을 유지하기 위한 dotfiles 저장소입니다.

## 구조

```
dotfiles/
└── claude/                    ← Claude Code 설정
    └── .claude/
        ├── CLAUDE.md          ← 글로벌 규칙 (언어, 작업 절차, 커밋 규칙 등)
        ├── settings.json      ← Claude Code 설정 (hooks, model, env)
        └── til/               ← TIL (Today I Learned) 노트
```

## 새 머신에 적용하기

### 1. 저장소 클론

```bash
git clone git@github.com:pigdog-code/dotfiles.git ~/work/dotfiles
```

### 2. 기존 파일 백업 (이미 파일이 있는 경우)

```bash
mv ~/.claude/CLAUDE.md ~/.claude/CLAUDE.md.bak
mv ~/.claude/settings.json ~/.claude/settings.json.bak
mv ~/.claude/til ~/.claude/til.bak
```

### 3. 심링크 생성

```bash
ln -s ~/work/dotfiles/claude/.claude/CLAUDE.md ~/.claude/CLAUDE.md
ln -s ~/work/dotfiles/claude/.claude/settings.json ~/.claude/settings.json
ln -s ~/work/dotfiles/claude/.claude/til ~/.claude/til
```

### 4. 확인

```bash
ls -la ~/.claude/CLAUDE.md ~/.claude/settings.json ~/.claude/til
# 화살표(->)로 연결된 심링크가 보이면 성공
```

### 5. Hooks 사용 여부 확인

`settings.json`에 포함된 hooks는 특정 머신 환경(스크립트 경로, 네트워크)에 의존합니다.
새 머신에 설치할 때 **hooks 사용 여부를 반드시 확인**하고, 사용하지 않을 경우 `~/.claude/settings.local.json`으로 비활성화합니다:

```bash
# hooks를 사용하지 않을 경우
cat > ~/.claude/settings.local.json << 'EOF'
{
  "hooks": {}
}
EOF
```

> **규칙**: `settings.local.json`은 머신별 로컬 설정으로, dotfiles에 포함하지 않습니다.

## 동기화 제외 항목

아래 파일들은 머신별 고유 데이터이므로 동기화하지 않습니다:

| 파일/폴더               | 이유                          |
|--------------------------|-------------------------------|
| `.credentials.json`      | 인증 토큰 (보안)              |
| `.mcp.json`              | MCP 서버 설정 (토큰 포함)     |
| `history.jsonl`          | 대화 히스토리                 |
| `projects/`              | 프로젝트별 설정/메모리        |
| `sessions/`              | 세션 데이터                   |
| `debug/`, `cache/`, `telemetry/` | 런타임 데이터          |

## 변경 후 동기화

설정을 수정한 후 다른 머신에 반영하려면:

```bash
cd ~/work/dotfiles
git add -A
git commit -m "변경 내용 설명"
git push
```

다른 머신에서:

```bash
cd ~/work/dotfiles
git pull
# 심링크가 이미 걸려 있으므로 자동 반영됨
```
