---
name: gpu-train
description: GPU 대여 + 모델 학습 자동화 — 프로젝트 분석·가격 검색을 병렬 에이전트로 처리하고, 가장 저렴한 GPU를 추천해 대여·실행·종료까지 대화로 진행.
argument-hint: "[파일명] [커스텀 실행 명령어]"
user-invokable: true
---

## When to invoke this skill

GPU가 필요한 모델 학습/생성 작업. "GPU 빌려서 학습해줘", "Colab 말고 더 싼 거", "vast.ai로 돌려줘", "가장 싼 GPU 찾아줘" 등.

## Preamble (run first)

```bash
_SKILL_DIR=~/.claude/skills/gpu-train
_STATE="$_SKILL_DIR/state"
mkdir -p "$_STATE"

# 버전 체크
_LOCAL_VER=$(cat "$_SKILL_DIR/VERSION" 2>/dev/null || echo "unknown")
_REMOTE_VER=$(curl -sf \
  "https://raw.githubusercontent.com/salmonco/gpu-skill/main/VERSION" \
  2>/dev/null || echo "")
if [ -n "$_REMOTE_VER" ] && [ "$_REMOTE_VER" != "$_LOCAL_VER" ]; then
  echo "UPGRADE_AVAILABLE: $_LOCAL_VER → $_REMOTE_VER"
else
  echo "VERSION: $_LOCAL_VER"
fi

# vastai CLI 감지
_VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1 \
  || command -v vastai 2>/dev/null || echo "")
echo "VASTAI_CLI: $([ -n "$_VASTAI" ] && echo 'found' || echo 'missing')"

# CLI 인증 상태 확인 (키를 직접 읽지 않음)
_VAST_USER=$("$_VASTAI" show user 2>/dev/null \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('username',''))" 2>/dev/null || echo "")
_KEYS_SET="no"; [ -n "$_VAST_USER" ] && _KEYS_SET="yes"
echo "KEYS_SET: $_KEYS_SET (user: ${_VAST_USER:-none})"

# 활성 인스턴스 상태
_ACTIVE=$(cat "$_STATE/.active_instance" 2>/dev/null || echo "")
echo "ACTIVE_INSTANCE: $([ -n "$_ACTIVE" ] && echo "$_ACTIVE" || echo 'none')"
```

`UPGRADE_AVAILABLE`이 출력되면 — 다른 것보다 먼저 AskUserQuestion으로 물어보기:

```
AskUserQuestion({
  questions: [{
    question: "gpu-train v<new> 나왔어 (현재 v<old>). 업데이트할까?",
    header: "업데이트",
    options: [
      { label: "지금 업데이트", description: "git pull로 최신 버전 설치 후 계속 진행" },
      { label: "나중에", description: "현재 버전으로 계속 진행" }
    ]
  }]
})
```

"지금 업데이트" 선택 시:
```bash
cd ~/.claude/skills/gpu-train && git pull origin main
echo "✅ 업데이트 완료 — $(cat VERSION)"
```
업데이트 후 preamble 재실행.

## Step 0: 활성 인스턴스 처리 [GATE]

`ACTIVE_INSTANCE`가 `none`이 아니면 — 새 작업 시작 전에 AskUserQuestion으로 물어보기:

```
AskUserQuestion({
  questions: [{
    question: "현재 실행 중인 인스턴스가 있어: <ID>. 어떻게 할까?",
    header: "활성 인스턴스",
    options: [
      { label: "결과 회수 + 종료", description: "결과 다운로드 후 인스턴스 삭제 (과금 중지)" },
      { label: "상태만 확인", description: "로그 확인 후 계속 실행" },
      { label: "무시하고 새 작업", description: "현재 인스턴스는 그대로 두고 새 작업 시작" }
    ]
  }]
})
```

"결과 회수 + 종료" 선택 → Step 4로 건너뜀.

## Step 1: 병렬 에이전트 — 프로젝트 분석 + 가격 검색

**두 에이전트를 동시에(single message) 실행:**

**Agent A — 프로젝트 분석** (`subagent_type: "general-purpose"`):
```
Read ~/.claude/skills/gpu-train/specialists/project.md and follow it exactly.
Return only the JSON output specified in that file.
Working directory: <current pwd>
```

**Agent B — 가격 검색** (`subagent_type: "general-purpose"`):
```
Read ~/.claude/skills/gpu-train/specialists/prices.md and follow it exactly.
MIN_VRAM_GB=16  (project agent 결과 나오기 전이므로 보수적으로 16 사용)
Return only the JSON output specified in that file.
```

가격 검색은 인증 불필요 — KEYS_SET 무관하게 항상 실행.

두 에이전트 완료 후 결과 합산:

```
📁 프로젝트: <project>
🤖 모델: <model> | 💾 필요 VRAM: <vram_gb>GB
📦 추천 파일: <recommended_file>
💰 최저가: <recommended_gpu> $<recommended_price>/h (offer: <recommended_id>)
→ 예상 비용: $<price>/h × ~30분 = ~$<estimated_cost>
```

그리고 AskUserQuestion으로 한 번에 물어보기:

```
AskUserQuestion({
  questions: [{
    question: "어떻게 진행할까?",
    header: "다음 단계",
    options: [
      { label: "추천대로 진행", description: "<recommended_gpu> $<price>/h — 바로 대여 시작" },
      { label: "파일/명령어 변경", description: "다른 파일 선택 또는 커스텀 실행 명령어 입력" },
      { label: "가격 목록 전체 보기", description: "상위 6개 GPU 옵션 비교" }
    ]
  }]
})
```

## Step 2: Vast.ai 인증 안내 [GATE: KEYS_SET=no, 대여 직전]

`KEYS_SET`이 `no`일 때만 실행. `yes`면 완전히 스킵.
가격 검색(Step 1)은 인증 없이 항상 실행되므로, 이 단계는 유저가 "추천대로 진행"을 선택한 후에만 진행.

아래 안내 후 AskUserQuestion으로 확인:

> **Vast.ai 인증 필요.** 터미널에서 직접 실행해줘 (키가 나를 거치지 않아):
> ```bash
> vastai set api-key <YOUR_API_KEY>
> ```
> API 키는 [vast.ai → 우상단 아이콘 → Account → API Key]에서 복사.

```
AskUserQuestion({
  questions: [{
    question: "터미널에서 `vastai set api-key KEY` 실행했어?",
    header: "Vast.ai 인증",
    options: [
      { label: "완료", description: "인증 확인 후 가격 검색 진행" },
      { label: "계정이 없어", description: "vast.ai 가입 링크 안내" },
      { label: "나중에 할게", description: "여기서 중단" }
    ]
  }]
})
```

"완료" 선택 시 → preamble 재실행해서 인증 확인 후 Agent B(가격 검색) 실행.

## Step 3: 대여 + 실행 에이전트 [승인 후]

**Agent C — 대여 & 실행** (`subagent_type: "general-purpose"`):
```
Read ~/.claude/skills/gpu-train/specialists/rent.md and follow it exactly.
OFFER_ID=<recommended_id from Step 1>
LOCAL_FILE=<recommended_file from Step 1>
REMOTE_CMD=<실행 명령어 — 사용자 확인 또는 기본값>
```

실행 명령어 기본값 (파일 종류에 따라):
- `.ipynb` → `pip install -q jupyter nbconvert && jupyter nbconvert --to notebook --execute <파일명> --output result.ipynb`
- `.py` → `pip install -q transformers peft torch && python <파일명>`

진행 상황 실시간 보여주기.

## Step 4: 완료 감지 + 회수 에이전트

작업 완료 신호 감지(로그 모니터링) 후:

**Agent D — 결과 회수 & 종료** (`subagent_type: "general-purpose"`):
```
Read ~/.claude/skills/gpu-train/specialists/collect.md and follow it exactly.
INSTANCE_ID=<from Step 3>
SSH_HOST=<from Step 3>
SSH_PORT=<from Step 3>
REMOTE_OUT=<결과 파일 경로>
LOCAL_OUT=~/Downloads/
```

## Voice

추천은 먼저, 묻는 건 꼭 필요할 때만. 비용은 항상 보여주기. 에이전트 진행 상황 짧게 보고.

## Safety

- 대여 전 **반드시 예상 비용 + 승인** — 자동 대여 절대 없음
- 완료/실패 어느 경우든 인스턴스 종료 안내
- API 키는 `vastai set api-key`로만 저장 — 스킬이 키를 직접 읽거나 저장하지 않음

## Adaptive Gating

| 조건 | 스킵 |
|------|------|
| KEYS_SET=yes | Step 2 완전 스킵 |
| KEYS_SET=no | Step 1(가격 검색) 정상 실행, Step 2는 대여 선택 시에만 |
| ACTIVE_INSTANCE≠none | AskUserQuestion으로 처리 방법 결정 |
| VASTAI_CLI=missing | 설치 안내 후 중단 |

## Completion Status

- **DONE**: 결과 다운로드 + 인스턴스 종료 확인
- **DONE_WITH_CONCERNS**: 완료했으나 일부 파일 누락
- **BLOCKED**: 인증 실패 / SSH 접속 불가 / 잔액 부족
