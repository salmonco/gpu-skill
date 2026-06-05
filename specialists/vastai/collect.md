# Vastai Collect & Shutdown Specialist

Implements platform-contract.md — collect.md interface.

## Input

- `INSTANCE_ID`: Vast.ai 인스턴스 ID
- `PLATFORM`: "vastai" (고정)
- `REMOTE_OUT`: 원격 결과 파일/폴더 경로
- `LOCAL_OUT`: 로컬 저장 경로 (기본: ~/Downloads/)
- `GPU_NAME`, `PRICE_PER_HOUR`, `PROJECT`, `FILE`: learnings 기록용
- `START_TIME`: 작업 시작 Unix timestamp

## Step 1: 작업 완료 확인

SSH 대신 vastai CLI로 로그 조회 (IP/포트 저장 불필요):

```bash
VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1 \
  || command -v vastai 2>/dev/null)

echo "최근 로그:"
"$VASTAI" logs "$INSTANCE_ID" 2>/dev/null | tail -20
```

완료 신호(`done`, `saved`, `epoch`, `완료`) → `JOB_RESULT=success`
에러 신호(`Error`, `Traceback`, `OOM`) → `JOB_RESULT=failed`, 오케스트레이터에 보고.

## Step 2: 결과 다운로드

```bash
LOCAL_OUT="${LOCAL_OUT:-$HOME/Downloads}"
mkdir -p "$LOCAL_OUT"

"$VASTAI" copy "$INSTANCE_ID:$REMOTE_OUT" "$LOCAL_OUT/" 2>/dev/null || \
  echo "⚠️ vastai copy 실패 — 수동 다운로드 필요"

echo "✅ 다운로드 완료: $LOCAL_OUT"
ls -lh "$LOCAL_OUT"
```

## Step 3: 인스턴스 종료

```bash
STATE_DIR=~/.claude/skills/gpu-train/state

"$VASTAI" destroy instance "$INSTANCE_ID"
rm -f "$STATE_DIR/.active_instance"
echo "✅ 인스턴스 종료 — 과금 중지"
```

## Step 4: GPU 이력 저장 (contract 필수)

```bash
STATE_DIR=~/.claude/skills/gpu-train/state
END_TIME=$(date +%s)
DURATION_MIN=$(( (END_TIME - ${START_TIME:-$END_TIME}) / 60 ))

python3 - <<PYEOF
import json, datetime
entry = {
    "date": datetime.date.today().isoformat(),
    "platform": "vastai",
    "offer_id": "${INSTANCE_ID}",
    "gpu": "${GPU_NAME:-unknown}",
    "price_per_hour": ${PRICE_PER_HOUR:-0},
    "project": "${PROJECT:-unknown}",
    "file": "${FILE:-unknown}",
    "result": "${JOB_RESULT:-unknown}",
    "duration_min": ${DURATION_MIN:-0},
    "notes": ""
}
with open("${STATE_DIR}/learnings.jsonl", "a") as f:
    f.write(json.dumps(entry, ensure_ascii=False) + "\n")
print(f"📝 이력 저장: {entry['platform']} {entry['gpu']} → {entry['result']}")
PYEOF
```

## Output Format

```json
{
  "platform": "vastai",
  "status": "success|partial|failed",
  "downloaded_to": "<LOCAL_OUT>",
  "instance_terminated": true,
  "notes": "<특이사항>"
}
```
