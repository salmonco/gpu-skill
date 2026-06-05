# gpu-skill

Claude Code 스킬 — 가장 저렴한 GPU를 실시간으로 찾아 빌리고, 모델 학습을 대화로 자동화.

## 지원 플랫폼
- Vast.ai (RTX 4090 ~$0.3/h)
- RunPod (예정)

## 설치 / 업데이트

신규 설치와 구버전 업데이트 모두 동일한 명령어:

```bash
rm -rf ~/.claude/skills/gpu-train
git clone https://github.com/salmonco/gpu-skill.git ~/.claude/skills/gpu-train
```

이후엔 `/gpu-train` 실행 시 새 버전이 있으면 자동으로 알려줘.

## 사용법
```
/gpu-train
```

## 구조
```
SKILL.md          # Claude가 읽는 스킬 지시서
VERSION           # 현재 버전
specialists/
  project.md      # 프로젝트 분석 에이전트
  prices.md       # GPU 가격 검색 에이전트
  rent.md         # 대여 + 실행 에이전트
  collect.md      # 결과 회수 + 종료 에이전트
bin/
  search-vast     # Vast.ai GPU 검색
  rent-instance   # 인스턴스 대여
  upload-and-run  # 파일 전송 + 원격 실행
  destroy-instance# 종료 (과금 중지)
state/            # 런타임 상태 (git 제외)
```
