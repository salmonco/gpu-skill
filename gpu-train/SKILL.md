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

# 상태 감지
_VAST_KEY=$(cat "$_STATE/.vastai_key" 2>/dev/null || echo "")
_KEYS_SET="no"; [ -n "$_VAST_KEY" ] && _KEYS_SET="yes"
echo "KEYS_SET: $_KEYS_SET"

_ACTIVE=$(cat "$_STATE/.active_instance" 2>/dev/null || echo "")
echo "ACTIVE_INSTANCE: $([ -n "$_ACTIVE" ] && echo "$_ACTIVE" || echo 'none')"

_VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1 \
  || command -v vastai 2>/dev/null || echo "")
echo "VASTAI_CLI: $([ -n "$_VASTAI" ] && echo 'found' || echo 'missing')"
```

## Step 0: 활성 인스턴스 처리 [GATE]

`ACTIVE_INSTANCE`가 `none`이 아니면 — 새 작업 시작 전에 먼저 보고:
"현재 실행 중인 인스턴스: `<ID>`. 결과 회수 + 종료할까, 아니면 상태만 확인할까?"

종료 원하면 → Step 4(collect)로 건너뜀.

## Step 1: 병렬 에이전트 — 프로젝트 분석 + 가격 검색

**두 에이전트를 동시에(single message) 실행:**

**Agent A — 프로젝트 분석** (`subagent_type: "general-purpose"`):
```
Read ~/.claude/skills/gpu-train/specialists/project.md and follow it exactly.
Return only the JSON output specified in that file.
Working directory: <current pwd>
```

**Agent B — 가격 검색** (`subagent_type: "general-purpose"`, `KEYS_SET=yes`일 때만):
```
Read ~/.claude/skills/gpu-train/specialists/prices.md and follow it exactly.
MIN_VRAM_GB=16  (project agent 결과 나오기 전이므로 보수적으로 16 사용)
Return only the JSON output specified in that file.
```

`KEYS_SET=no`이면 Agent B 스킵 → Step 2(API 키 설정) 먼저.

두 에이전트 완료 후 결과 합산:

```
📁 프로젝트: <project>
🤖 모델: <model> | 💾 필요 VRAM: <vram_gb>GB
📦 추천 파일: <recommended_file>
💰 최저가: <recommended_gpu> $<recommended_price>/h (offer: <recommended_id>)
→ 예상 비용: $<price>/h × ~30분 = ~$<price*0.5>
```

**한 번에 물어:**

Options:
- A) 추천대로 진행
- B) 다른 파일/명령어 지정
- C) 가격 목록 전체 보기

## Step 2: API 키 설정 [GATE: KEYS_SET=no]

`KEYS_SET`이 `no`일 때만 실행. 이미 `yes`면 완전히 스킵.

> Vast.ai API 키 필요. [vast.ai → 우상단 아이콘 → API Key]에서 복사해줘.

받으면:
```bash
echo "<KEY>" > ~/.claude/skills/gpu-train/state/.vastai_key
chmod 600 ~/.claude/skills/gpu-train/state/.vastai_key
echo "✅ 저장 완료"
```

→ Step 1의 Agent B(가격 검색)를 이제 실행.

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
- API 키 `state/`에만 저장 (git 제외)

## Adaptive Gating

| 조건 | 스킵 |
|------|------|
| KEYS_SET=yes | Step 2 완전 스킵 |
| ACTIVE_INSTANCE≠none | Step 1~3 스킵, Step 4로 바로 |
| VASTAI_CLI=missing | 설치 안내 후 중단 |

## Completion Status

- **DONE**: 결과 다운로드 + 인스턴스 종료 확인
- **DONE_WITH_CONCERNS**: 완료했으나 일부 파일 누락
- **BLOCKED**: API 키 없음 / SSH 접속 불가 / 잔액 부족
