# gpu-skill

Claude Code 스킬 — 가장 저렴한 GPU를 실시간으로 찾아 빌리고, 모델 학습을 대화로 자동화.

## 지원 플랫폼
- Vast.ai (RTX 4090 ~$0.3/h)
- RunPod (예정)

## 설치

```bash
git clone https://github.com/salmonco/gpu-skill.git ~/.claude/skills/gpu-skill
```

## 사용법
```
/gpu-train
```

## 구조
```
gpu-train/
  SKILL.md          # Claude가 읽는 스킬 지시서
  bin/
    search-vast     # Vast.ai GPU 검색
    rent-instance   # 인스턴스 대여
    upload-and-run  # 파일 전송 + 원격 실행
    destroy-instance# 종료 (과금 중지)
  state/            # API 키, 활성 인스턴스 (git 제외)
```
