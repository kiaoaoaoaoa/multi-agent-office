# 🏢 GitHub Actions 기반 멀티 AI 에이전트 오케스트레이터

GitHub를 중앙 오피스로 사용하여 Claude, Gemini, Codex를 부서처럼 동시에 운영하는 시스템.

## 구조

```
[GitHub Repo = 중앙 오피스]
    │
    ├── Issues (작업 지시서)
    │   ├── agent:claude  → 서류 담당 부서
    │   ├── agent:gemini  → 대외 커뮤니케이션 부서
    │   └── agent:codex   → 정보수집/리서치 부서
    │
    ├── agents/state.json (실시간 공유 상태)
    ├── agents/results/   (각 부서 작업 결과물)
    └── .github/workflows/
        ├── orchestrator.yml   (5분마다 스캔 + 작업 할당)
        ├── agent-claude.yml   (Claude API 호출)
        ├── agent-gemini.yml   (Gemini API 호출)
        └── agent-codex.yml    (OpenAI API 호출)
```

## 작동 방식

```
1. 당신이 Issue 생성 + 라벨(agent:claude 등) 부착
2. Orchestrator가 5분마다 스캔 → 미할당 작업 자동 배정
3. 해당 Agent Workflow 트리거 → AI API 호출 → 결과물 생성
4. agents/state.json 실시간 업데이트 → 모든 에이전트가 최신 상태 확인
5. depends-on 태그로 부서 간 의존성 관리
```

## 설정 방법

### 1. GitHub 레포 생성 후 푸시
```bash
gh repo create multi-agent-office --public --push
```

### 2. API 키 등록 (GitHub Secrets)
Settings → Secrets and variables → Actions → New repository secret:

| Secret 이름 | 값 |
|---|---|
| `ANTHROPIC_API_KEY` | Claude API 키 |
| `GEMINI_API_KEY` | Gemini API 키 |
| `OPENAI_API_KEY` | OpenAI API 키 |

### 3. 권한 설정
Settings → Actions → General → Workflow permissions:
- `Read and write permissions` 선택
- `Allow GitHub Actions to create and approve pull requests` 체크

## 사용법

### 작업 지시
```bash
# Claude에게 계약서 작성 지시
gh issue create \
  --title "ABC사 계약서 초안 작성" \
  --label "agent:claude" \
  --body "## 작업 내용
ABC사와의 물품 공급 계약서 초안을 작성해주세요.
- 계약 기간: 2026.07 ~ 2027.06
- 월 공급 금액: 5,000만원

## 의존성
depends-on: agent:codex (ABC사 신용정보 필요)"
```

### 동시 작업 예시
```bash
# 부서 3곳에 동시 지시
gh issue create -t "거래처 뉴스레터 초안" -l "agent:gemini" -b "..."
gh issue create -t "신규 거래처 리서치" -l "agent:codex" -b "..."
gh issue create -t "월간 실적 보고서" -l "agent:claude" -b "..."
```

### 진행상황 확인
```bash
# 대시보드
gh issue list --state open --json title,labels,assignees

# 특정 부서 작업 확인
gh issue list --label "agent:claude" --state open

# 공유 상태 확인
cat agents/state.json | jq .
```

## 부서 간 협업 예시

```
1. 당신: "신규 거래처랑 계약해" → Issue 3개 생성
   ├── #1 "거래처 신용조사" [agent:codex]
   ├── #2 "컨택 이메일 초안" [agent:gemini]
   └── #3 "계약서 초안" [agent:claude] depends-on: codex, gemini

2. Codex: 신용정보 수집 완료 → agents/results/codex-*.md
3. Gemini: 이메일 초안 작성 완료 → #3에 "@agent:claude 준비됨" 코멘트
4. Claude: codex+gemini 결과 확인 → 계약서 작성 시작
5. 모든 결과가 Issue 코멘트 + 레포 파일로 기록
```

## 확장

에이전트 추가는 간단:
1. `.github/workflows/agent-xxx.yml` 복사해서 새 AI API 호출로 수정
2. `agents/prompts/xxx.md`에 역할 정의
3. GitHub Secrets에 해당 API 키 추가
4. `orchestrator.yml`에 새 라벨 등록
