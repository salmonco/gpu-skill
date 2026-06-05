# Rent & Run Specialist

인스턴스를 대여하고 파일을 전송해 작업을 실행한다.

## Input (오케스트레이터에서 전달)

- `OFFER_ID`: Vast.ai offer ID
- `LOCAL_FILE`: 전송할 로컬 파일/폴더 경로
- `REMOTE_CMD`: 원격에서 실행할 명령어

## Step 1: 대여

```bash
STATE_DIR=~/.claude/skills/gpu-train/state
VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1)
API_KEY=$(cat "$STATE_DIR/.vastai_key")
"$VASTAI" set api-key "$API_KEY" >/dev/null 2>&1

RESULT=$("$VASTAI" create instance "$OFFER_ID" \
  --image pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime \
  --disk 40 2>&1)
echo "$RESULT"

# 인스턴스 ID 저장
INSTANCE_ID=$(echo "$RESULT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('new_contract',''))" 2>/dev/null || \
              echo "$RESULT" | grep -oE '"id":[0-9]+' | head -1 | grep -oE '[0-9]+')
echo "vast:$INSTANCE_ID" > "$STATE_DIR/.active_instance"
echo "INSTANCE_ID: $INSTANCE_ID"
```

## Step 2: SSH 준비 대기 (최대 3분)

```bash
for i in $(seq 1 18); do
  sleep 10
  INFO=$("$VASTAI" show instance "$INSTANCE_ID" 2>/dev/null)
  SSH_HOST=$(echo "$INFO" | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(d.get('ssh_host',''))" 2>/dev/null)
  SSH_PORT=$(echo "$INFO" | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(d.get('ssh_port',''))" 2>/dev/null)
  [ -n "$SSH_HOST" ] && [ "$SSH_HOST" != "null" ] && break
  echo "대기 중... ($((i*10))s)"
done
echo "SSH_HOST: $SSH_HOST"
echo "SSH_PORT: $SSH_PORT"
```

## Step 3: 파일 전송 + 실행

```bash
# 파일 전송
scp -P "$SSH_PORT" -o StrictHostKeyChecking=no -o ConnectTimeout=30 \
  -r "$LOCAL_FILE" "root@${SSH_HOST}:~/"

# 원격 실행 (백그라운드 + 로그 캡처)
ssh -p "$SSH_PORT" -o StrictHostKeyChecking=no "root@${SSH_HOST}" \
  "nohup bash -c '$REMOTE_CMD' > ~/run.log 2>&1 &"

echo "실행 시작됨. 로그: ~/run.log"
echo "HOST: $SSH_HOST  PORT: $SSH_PORT  INSTANCE: $INSTANCE_ID"
```

## Output Format

```json
{
  "instance_id": "<ID>",
  "ssh_host": "<host>",
  "ssh_port": <port>,
  "status": "running|failed",
  "log_path": "~/run.log"
}
```
