# Platform Contract

새 플랫폼 추가 시 이 계약을 구현해야 한다.
각 플랫폼은 `specialists/<platform>/` 디렉토리 아래 세 파일을 구현한다.

## prices.md

**Input**
- `MIN_VRAM_GB`: 최소 VRAM (기본 16)
- `MAX_PRICE`: 시간당 최대 가격 (기본 2.0)

**Output JSON** (반드시 이 스키마 준수)
```json
{
  "platform": "<platform>",
  "offers": [
    {
      "id": "<offer_id>",
      "gpu": "<gpu_name>",
      "vram_gb": 24,
      "price_per_hour": 0.29,
      "reliability": 0.99,
      "location": "US"
    }
  ],
  "recommended_id": "<offer_id>",
  "recommended_gpu": "<gpu_name>",
  "recommended_price": 0.29
}
```

실패 시: `{ "platform": "<platform>", "error": "<이유>", "offers": [] }`

---

## rent.md

**Input**
- `OFFER_ID`: 플랫폼 offer ID
- `LOCAL_FILE`: 전송할 로컬 파일 경로
- `REMOTE_CMD`: 원격 실행 명령어

**Output JSON**
```json
{
  "platform": "<platform>",
  "instance_id": "<id>",
  "status": "running|failed",
  "log_path": "~/run.log"
}
```

**state 기록** (오케스트레이터가 읽음)
```bash
echo "<platform>:<instance_id>" > ~/.claude/skills/gpu-train/state/.active_instance
```

---

## collect.md

**Input**
- `INSTANCE_ID`: 인스턴스 ID
- `PLATFORM`: 플랫폼명 (vastai / runpod / ...)
- `REMOTE_OUT`: 원격 결과 경로
- `LOCAL_OUT`: 로컬 저장 경로
- `GPU_NAME`, `PRICE_PER_HOUR`, `PROJECT`, `FILE`, `START_TIME`: learnings 기록용

**Output JSON**
```json
{
  "platform": "<platform>",
  "status": "success|partial|failed",
  "downloaded_to": "<LOCAL_OUT>",
  "instance_terminated": true,
  "notes": "<특이사항>"
}
```

**learnings 기록** (반드시 포함)
```json
{ "date": "...", "platform": "<platform>", "offer_id": "...", "gpu": "...",
  "price_per_hour": 0.0, "project": "...", "file": "...",
  "result": "success|failed", "duration_min": 0, "notes": "" }
```

---

## 새 플랫폼 추가 체크리스트

- [ ] `specialists/<platform>/prices.md` — 위 Output JSON 스키마 준수
- [ ] `specialists/<platform>/rent.md` — state에 `<platform>:<id>` 형식으로 기록
- [ ] `specialists/<platform>/collect.md` — learnings.jsonl에 platform 필드 포함
- [ ] `SKILL.md` preamble에 인증 체크 추가
- [ ] `platform-contract.md` 체크리스트에 플랫폼 등록
