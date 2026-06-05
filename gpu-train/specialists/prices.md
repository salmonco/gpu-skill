# Price Search Specialist

Vast.ai에서 실시간 GPU 가격을 검색해 최저가 목록을 반환한다.
가격 검색은 인증 불필요 — API 키 없이도 동작.

## Input

- `MIN_VRAM_GB`: 필요 최소 VRAM (기본 16)
- `MAX_PRICE`: 시간당 최대 가격 (기본 2.0)

## Task

```bash
VASTAI=$(find "$HOME" -name vastai -path "*/bin/vastai" 2>/dev/null | head -1 \
  || command -v vastai 2>/dev/null || echo "")

if [ -z "$VASTAI" ]; then
  echo '{"error": "vastai CLI 없음 — pip install vastai 로 설치"}'
  exit 0
fi

# 검색: 인증 불필요, 신뢰도 높고 저렴한 순
"$VASTAI" search offers \
  "gpu_ram>=${MIN_VRAM_GB:-16} reliability>0.97 dph_total<${MAX_PRICE:-2.0} num_gpus=1 inet_up>200" \
  --order dph_total \
  --limit 6 \
  2>/dev/null
```

## Output Format

검색 결과를 아래 JSON으로 파싱해 반환:

```json
{
  "offers": [
    {
      "rank": 1,
      "id": "<offer_id>",
      "gpu": "<gpu_name>",
      "vram_gb": <vram>,
      "price_per_hour": <price>,
      "reliability": <0~1>,
      "location": "<country>"
    }
  ],
  "recommended_id": "<가장 저렴하고 신뢰도 높은 offer_id>",
  "recommended_gpu": "<gpu_name>",
  "recommended_price": <price>
}
```

검색 실패 시: `{"error": "<이유>", "offers": []}`
