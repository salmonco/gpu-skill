---
name: gpu-train
description: GPU 대여 + 모델 학습 자동화 — 프로젝트 컨텍스트를 자동 감지하고 가장 저렴한 GPU를 추천해 대여·실행·종료까지 대화로 진행.
argument-hint: "[플랫폼] [작업 설명]"
user-invokable: true
---

## When to invoke this skill

GPU가 필요한 모델 학습/생성 작업. "GPU 빌려서 학습해줘", "Colab 말고 더 싼 거 없어?", "vast.ai로 돌려줘" 등.

## Preamble (run first)

```bash
_SKILL_DIR=~/.claude/skills/gpu-train
_BIN="$_SKILL_DIR/bin"
_STATE="$_SKILL_DIR/state"
mkdir -p "$_STATE"

# 1. API 키 상태
_VAST_KEY=$(cat "$_STATE/.vastai_key" 2>/dev/null || echo "")
_KEYS_SET="no"; [ -n "$_VAST_KEY" ] && _KEYS_SET="yes"
echo "KEYS_SET: $_KEYS_SET"

# 2. 활성 인스턴스
_ACTIVE=$(cat "$_STATE/.active_instance" 2>/dev/null || echo "")
echo "ACTIVE_INSTANCE: $([ -n "$_ACTIVE" ] && echo "$_ACTIVE" || echo 'none')"

# 3. 프로젝트 컨텍스트 자동 감지
_PROJECT=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || basename "$PWD")
echo "PROJECT: $_PROJECT"

# 4. 실행할 파일 후보 감지 (노트북/스크립트)
_NOTEBOOKS=$(find . -name "colab_*.ipynb" -o -name "train_*.py" 2>/dev/null | head -5 | tr '\n' ' ')
echo "TRAIN_FILES: ${_NOTEBOOKS:-none}"

# 5. 모델 크기로 VRAM 추정
_VRAM_HINT=16
_MODEL_NAME="(미감지)"
if grep -rq "ACE-Step\|ace.step\|acestep" . --include="*.py" --include="*.ipynb" 2>/dev/null; then
  _MODEL_NAME="ACE-Step 1.5 (3.5B)"; _VRAM_HINT=16
elif grep -rq "musicgen-medium\|musicgen-large" . --include="*.py" --include="*.ipynb" 2>/dev/null; then
  _MODEL_NAME="MusicGen medium/large"; _VRAM_HINT=24
elif grep -rq "musicgen-small\|musicgen" . --include="*.py" --include="*.ipynb" 2>/dev/null; then
  _MODEL_NAME="MusicGen small"; _VRAM_HINT=8
fi
echo "MODEL_DETECTED: $_MODEL_NAME"
echo "VRAM_HINT_GB: $_VRAM_HINT"

# 6. 추천 GPU 결정
_REC_GPU="RTX 4090"
_REC_REASON="24GB bf16 지원, ~\$0.3/h"
[ "$_VRAM_HINT" -ge 40 ] && _REC_GPU="A100 40GB" && _REC_REASON="40GB, ~\$1.5/h"
echo "REC_GPU: $_REC_GPU"
echo "REC_REASON: $_REC_REASON"

# 7. vastai CLI 위치
_VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1 || command -v vastai 2>/dev/null || echo "")
echo "VASTAI_CLI: $([ -n "$_VASTAI" ] && echo "found: $_VASTAI" || echo 'missing')"
```

## Step 0: 활성 인스턴스 있으면 먼저 보고

`ACTIVE_INSTANCE`가 `none`이 아니면 멈추고 보고:
"현재 실행 중인 인스턴스: `<ID>`. 결과 다운로드 + 종료 할까, 아니면 상태 확인할까?"

## Step 1: 컨텍스트 요약 + 추천 (묻기 전에 먼저 제시)

감지 결과를 바탕으로 **추천을 먼저** 말해:

```
📁 프로젝트: <PROJECT>
🤖 감지된 모델: <MODEL_DETECTED>
📦 학습 파일: <TRAIN_FILES>
💾 필요 VRAM: <VRAM_HINT_GB>GB 이상
→ 추천 GPU: <REC_GPU> (<REC_REASON>)
```

그 다음 **한 번에** 물어:

Options:
- A) 추천대로 진행 (REC_GPU)
- B) 다른 GPU / 조건 직접 지정
- C) 가격 목록 먼저 보고 결정

## Step 2: API 키 설정 (KEYS_SET=no 일 때만)

`KEYS_SET`이 `no`이면:

> Vast.ai API 키가 필요해. [vast.ai → 계정 → API Key]에서 복사해줘.

키 받으면:
```bash
echo "<KEY>" > "$_STATE/.vastai_key"
chmod 600 "$_STATE/.vastai_key"
echo "✅ API 키 저장됨"
```

## Step 3: 가격 검색

```bash
"$_BIN/search-vast" <VRAM_HINT_GB>
```

결과를 표로 정리:
| # | GPU | VRAM | $/h | 신뢰도 |
|---|-----|:---:|:---:|:---:|

"1번 추천 — 진행할까?" 바로 제안.

## Step 4: 비용 예상 + 승인 (절대 생략 불가)

```
예상 비용: $<PRICE>/h × ~<TIME>분 = ~$<TOTAL>
```

> 진행할까?

Options:
- A) 네, 대여
- B) 아니오, 취소

A이면:
```bash
"$_BIN/rent-instance" vast <OFFER_ID>
```

## Step 5: 파일 전송 + 실행

SSH 정보 나올 때까지 대기(최대 2분) 후:
```bash
"$_BIN/upload-and-run" <INSTANCE_ID> <LOCAL_FILE> "<REMOTE_CMD>"
```

진행 상황 실시간 보여주기.

## Step 6: 완료 → 결과 회수 → 종료

```bash
# 결과 다운로드
scp -P <PORT> -o StrictHostKeyChecking=no root@<HOST>:<REMOTE_OUT> ~/Downloads/

# 인스턴스 종료
"$_BIN/destroy-instance" vast <INSTANCE_ID>
```

"✅ 완료. 결과: `~/Downloads/<파일>`. 인스턴스 종료 — 과금 중지." 로 마무리.

## Voice

직접적, 짧게. 추천은 먼저, 묻는 건 꼭 필요할 때만. 비용은 항상 보여주기.

## Safety

- 대여 전 **반드시 예상 비용 보여주고 승인** — 자동 대여 절대 없음
- 학습 완료/실패/중단 어느 경우든 인스턴스 종료 안내
- API 키는 `state/`에만 저장 (git 제외, chmod 600)

## Completion Status

- **DONE**: 결과 다운로드 + 인스턴스 종료 확인
- **DONE_WITH_CONCERNS**: 완료했으나 일부 누락
- **BLOCKED**: API 키 없음 / 접속 불가 / 잔액 부족
