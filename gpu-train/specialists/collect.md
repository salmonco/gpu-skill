# Collect & Shutdown Specialist

작업 완료 후 결과를 다운로드하고 인스턴스를 종료한다.

## Input

- `INSTANCE_ID`: Vast.ai 인스턴스 ID
- `SSH_HOST`, `SSH_PORT`: 접속 정보
- `REMOTE_OUT`: 원격 결과 파일/폴더 경로
- `LOCAL_OUT`: 로컬 저장 경로 (기본: ~/Downloads/)

## Step 1: 작업 완료 확인

```bash
# 로그에서 완료/에러 감지
RESULT=$(ssh -p "$SSH_PORT" -o StrictHostKeyChecking=no "root@${SSH_HOST}" \
  "tail -5 ~/run.log" 2>/dev/null)
echo "마지막 로그:"
echo "$RESULT"
```

완료 신호(`완료`, `done`, `saved`, `epoch 6/6` 등) 감지 후 진행.
에러 신호(`Error`, `Traceback`, `OOM`) 감지 시 → 오케스트레이터에 보고.

## Step 2: 결과 다운로드

```bash
LOCAL_OUT="${LOCAL_OUT:-$HOME/Downloads}"
mkdir -p "$LOCAL_OUT"

scp -P "$SSH_PORT" -o StrictHostKeyChecking=no -r \
  "root@${SSH_HOST}:${REMOTE_OUT}" "$LOCAL_OUT/"

echo "✅ 다운로드 완료: $LOCAL_OUT"
ls -lh "$LOCAL_OUT"
```

## Step 3: 인스턴스 종료

```bash
STATE_DIR=~/.claude/skills/gpu-train/state
VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1)
API_KEY=$(cat "$STATE_DIR/.vastai_key")
"$VASTAI" set api-key "$API_KEY" >/dev/null 2>&1

"$VASTAI" destroy instance "$INSTANCE_ID"
rm -f "$STATE_DIR/.active_instance"
echo "✅ 인스턴스 종료 — 과금 중지"
```

## Output Format

```json
{
  "status": "success|partial|failed",
  "downloaded_to": "<LOCAL_OUT>",
  "instance_terminated": true,
  "notes": "<특이사항>"
}
```
