# prompt-workshop

AI 업무 프롬프트를 자동으로 개선하는 워크숍 실습 저장소입니다.

[Karpathy의 autoresearch](https://github.com/karpathy/autoresearch)에서 영감을 받았습니다.
autoresearch에서 사람이 `program.md`만 다듬고 AI가 코드를 자율적으로 개선하듯,
이 저장소에서는 사람이 `program.md`만 다듬고 **AI가 업무 프롬프트를 자율적으로 개선**합니다.

## 구조

```
prompt-workshop/
├── program.md                # AI에게 주는 메타 지시서 (이것만 다듬습니다)
├── prompts/                  # AI가 개선하는 업무 프롬프트 (최소 버전에서 시작)
│   ├── code-review.md        # 코드 리뷰
│   ├── test-case.md          # TC 설계
│   ├── spec-review.md        # 기획서 리뷰
│   └── document-draft.md     # 문서 초안 작성
├── evaluation/               # 고정된 평가 기준 (수정하지 않습니다)
│   ├── code-review.md
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
| `prepare.py` — 고정된 평가 함수 | `evaluation/*.md` — 고정된 평가 기준 (측정 가능한 채점 기준) |
| `val_bpb` — 연속적 수치 지표 | 체크리스트 점수 — evaluation 파일에 정의된 기준으로 채점 |

## 사용법

### 1. Fork

이 저장소를 본인 GitHub 계정으로 fork합니다.

### 2. Gemini에 연결

Gemini 웹에서 fork한 저장소를 연결합니다. (+ 버튼 → 코드 가져오기)

### 3. 수동 1사이클 (베이스라인)

```
이 저장소의 구조를 먼저 파악해줘.

prompts/code-review.md를 프롬프트로 사용해서
samples/sample-code-1.md의 코드를 리뷰해줘.

리뷰 결과를 evaluation/code-review.md 기준으로 채점해줘.
```

최소 프롬프트("코드를 리뷰해주세요")로 시작하면 점수가 낮게 나옵니다. 이게 베이스라인입니다.

### 4. 자동 개선 사이클

Canvas 모드에서 자동 개선 앱을 만들어 사이클을 돌리면, AI가 프롬프트를 자동으로 개선합니다.
(상세 가이드: [agentkit 워크숍 심화](https://agentkit.vercel.app/workshop/advanced/cycle-lab))

### 5. 검증

개선된 프롬프트가 다른 코드에서도 잘 작동하는지, 나머지 4개 샘플 코드로 검증합니다.

### 6. 본인 맥락에 맞게 수정

`prompts/` 안의 파일을 본인 팀·업무에 맞게 수정합니다.

## 실무 적용

워크숍 후 돌아가서:
- `samples/`에 **실제 팀 코드**를 넣고
- `prompts/`를 **실제 팀 컨벤션**으로 수정하고
- AI에 연결해서 계속 돌리면

→ 팀 맞춤형 AI 프롬프트가 자동으로 만들어지는 구조가 됩니다.
