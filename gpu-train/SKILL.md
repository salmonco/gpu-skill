# gpu-train: GPU 대여 + 모델 학습 자동화

가장 저렴한 GPU를 실시간으로 찾아 빌리고, 파일 전송 → 학습 → 결과 회수 → 종료까지 대화로 진행.

## Preamble (run first)

```bash
_SKILL_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}" 2>/dev/null || echo ~/.claude/skills/gpu-train)" && pwd)"
_BIN="$_SKILL_DIR/bin"
_STATE="$_SKILL_DIR/state"
mkdir -p "$_STATE"

# API 키 확인
_VAST_KEY=$(cat "$_STATE/.vastai_key" 2>/dev/null || echo "")
_RUNPOD_KEY=$(cat "$_STATE/.runpod_key" 2>/dev/null || echo "")
_KEYS_SET="no"
[ -n "$_VAST_KEY" ] || [ -n "$_RUNPOD_KEY" ] && _KEYS_SET="yes"

echo "KEYS_SET: $_KEYS_SET"
echo "VAST_KEY: $([ -n "$_VAST_KEY" ] && echo 'set' || echo 'missing')"
echo "RUNPOD_KEY: $([ -n "$_RUNPOD_KEY" ] && echo 'set' || echo 'missing')"

# 현재 실행 중인 인스턴스 확인
_ACTIVE=$(cat "$_STATE/.active_instance" 2>/dev/null || echo "")
echo "ACTIVE_INSTANCE: $([ -n "$_ACTIVE" ] && echo "$_ACTIVE" || echo 'none')"
```

## Setup (최초 1회)

`KEYS_SET`이 `no`이면:

> GPU 대여를 위해 API 키가 필요해. 어떤 플랫폼을 쓸까?

Options:
- A) Vast.ai (RTX 4090 ~$0.3/h, 가장 저렴)
- B) RunPod (안정적, ~$0.5/h)
- C) 둘 다 (가격 비교해서 더 싼 곳)

선택 후 API 키 입력받아 저장:
```bash
# Vast.ai 선택 시
echo "<API_KEY>" > "$_STATE/.vastai_key"
chmod 600 "$_STATE/.vastai_key"

# RunPod 선택 시
echo "<API_KEY>" > "$_STATE/.runpod_key"
chmod 600 "$_STATE/.runpod_key"
```

## Core Workflow

### Step 1. 작업 파악
사용자에게 묻기:
1. 어떤 학습? (모델명, 프레임워크)
2. VRAM 얼마나 필요? (모르면 내가 추정)
3. 예상 학습 시간?
4. 전송할 파일/폴더?

### Step 2. GPU 검색 + 비교
```bash
# Vast.ai 검색
"$_BIN/search-vast" <min_vram_gb>

# RunPod 검색 (키 있을 때)
"$_BIN/search-runpod" <gpu_type>
```

결과를 표로 보여주고 추천:
| 플랫폼 | GPU | VRAM | 가격/h | 신뢰도 | 추천 |
|--------|-----|:---:|:---:|:---:|:---:|

### Step 3. 승인 받고 대여
```bash
"$_BIN/rent-instance" <platform> <offer_id>
```
인스턴스 ID를 `$_STATE/.active_instance`에 저장.

### Step 4. 파일 전송 + 학습 시작
```bash
"$_BIN/upload-and-run" <instance_id> <local_path> <remote_cmd>
```

### Step 5. 완료 후 결과 회수 + 종료
```bash
"$_BIN/download-results" <instance_id> <remote_path> <local_path>
"$_BIN/destroy-instance" <instance_id>
```
`$_STATE/.active_instance` 삭제.

## Safety

- 대여 전 항상 **예상 비용 먼저 보여주고 승인** 받기
- 학습 완료/실패 어느 쪽이든 인스턴스 종료 안내
- API 키는 `state/` 에만 저장 (git 제외)

## Completion Status

- **DONE**: 결과 회수 + 인스턴스 종료 확인
- **DONE_WITH_CONCERNS**: 완료했으나 일부 실패
- **BLOCKED**: API 키 없음 / 인스턴스 접속 불가
