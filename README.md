# prompt-workshop

AI 업무 프롬프트를 자동으로 개선하는 워크숍 실습 저장소입니다.

[Karpathy의 autoresearch](https://github.com/karpathy/autoresearch)에서 영감을 받았습니다.
autoresearch에서 사람이 `program.md`만 다듬고 AI가 코드를 자율적으로 개선하듯,
이 저장소에서는 사람이 `program.md`만 다듬고 **AI가 업무 프롬프트를 자율적으로 개선**합니다.

## 구조

```
prompt-workshop/
├── program.md                # AI에게 주는 메타 지시서 (이것만 다듬습니다)
├── prompts/                  # AI가 개선하는 업무 프롬프트
│   ├── code-review.md        # 코드 리뷰
│   ├── test-case.md          # TC 설계
│   ├── spec-review.md        # 기획서 리뷰
│   └── document-draft.md     # 문서 초안 작성
├── evaluation/               # 고정된 평가 기준 (수정하지 않습니다)
│   ├── code-review.md        # 10개 항목, 각 1~5점, 만점 50점
│   ├── test-case.md
│   ├── spec-review.md
│   └── document-draft.md
└── samples/                  # 실습용 샘플 코드 5개
    ├── sample-code-1.md      # 출금 요청 처리
    ├── sample-code-2.md      # 자동 매수 예약
    ├── sample-code-3.md      # 주소 화이트리스트
    ├── sample-code-4.md      # 거래 내역 조회/CSV
    └── sample-code-5.md      # 알림 설정/발송
```

## autoresearch와의 대응

| autoresearch | prompt-workshop |
|---|---|
| `program.md` — AI의 연구 방향 지시 | `program.md` — AI의 프롬프트 개선 방향 지시 |
| `train.py` — AI가 수정하는 코드 | `prompts/*.md` — AI가 개선하는 프롬프트 |
| `prepare.py` — 고정된 평가 함수 | `evaluation/*.md` — 고정된 평가 기준 (10항목 x 5점 = 50점) |
| `val_bpb` — 낮을수록 좋은 단일 지표 | 체크리스트 점수 — 높을수록 좋음 (만점 50점, 도달 사실상 불가능) |

## 사용법

### 1. Fork

이 저장소를 본인 GitHub 계정으로 fork합니다.

### 2. 본인 맥락에 맞게 수정

GitHub 웹에서 `prompts/` 안의 파일을 본인 팀·업무에 맞게 수정합니다.
예: 팀명, 기술 스택, 도메인 규칙, 컨벤션 등을 본인 것으로 바꾸기.

### 3. Gemini에 연결

Gemini 웹에서 fork한 저장소를 연결합니다.

### 4. 자동 개선 사이클 실행

```
이 저장소의 program.md를 읽고 지시대로 따라해줘.
prompts/code-review.md를 개선 대상으로,
evaluation/code-review.md를 평가 기준으로,
samples/sample-code-1.md를 테스트 시나리오로 사용해서
자동 개선 사이클을 3회 실행해줘.
```

### 5. 검증

개선된 프롬프트가 다른 코드에서도 잘 작동하는지, 나머지 4개 샘플 코드로 검증합니다.
(autoresearch에서 validation set으로 측정하는 것과 같은 원리)

### 6. program.md 다듬기

AI의 개선 방향이 마음에 안 들면, `program.md`를 수정해서 다시 돌립니다.

## 실무 적용

워크숍 후 돌아가서:
- `samples/`에 **실제 팀 코드**를 넣고
- `prompts/`를 **실제 팀 컨벤션**으로 수정하고
- AI에 연결해서 계속 돌리면

→ 팀 맞춤형 AI 프롬프트가 자동으로 만들어지는 구조가 됩니다.
