# AI-assisted Comment Moderation Policy Lab  
_(공적영역과 테크노사이언스 · 게시판 댓글 규제 실험)_

이 저장소는 **대학 게시판 댓글 규제 정책을 어떻게 AI로 실험·개선할 수 있는지**를 보여주는 작은 실험실입니다.

핵심 질문은 **“댓글 규제 정책을 만드는 과정에서 AI를 어떻게 활용할 수 있는가?”** 입니다.

- AI에게 **여러 가지 댓글 규제 정책(A/B/C)을 설명**해 주고,
- 실제 게시판 댓글에 적용해 보면서 **차단/경고/허용 결과를 시뮬레이션**하고,
- **사람이 ‘이 경우엔 이렇게 처리하는 게 더 좋다’**라는 피드백을 남기면,
- 그 피드백을 바탕으로 **정책 v2 초안을 다시 AI가 제안**하게 만들고,
- 이 v2 정책을 다시 LLM 입력으로 넣어 **정책 결정 과정을 순환(iteration)** 시켜볼 수 있습니다.

즉, 이 프로젝트는 **“AI가 정책을 대신 결정한다”**가 아니라,  
**“정책 결정 과정을 설계하고 돌려보는 도구로 AI를 쓸 수 있는가?”**를 실험하는 데 목적이 있습니다.

---

## 전체 파이프라인 한눈에 보기

폴더 구조 (가정):

```text
project_root/
  ├─ data/
  │   ├─ raw_everytime.txt              # 게시판에서 긁어온 원시 텍스트
  │   └─ comments.csv                   # 전처리된 댓글/원글 데이터
  ├─ results/
  │   ├─ comments_with_policy_results.csv     # A/B/C 정책 시뮬레이션 결과
  │   ├─ disagreement_cases_with_text.csv     # 정책 간 불일치 + 원문 텍스트
  │   ├─ disagreement_with_human_template.csv # 사람이 라벨링할 템플릿
  │   ├─ disagreement_with_human_filled.csv   # 사람이 채운 피드백 (예시 이름)
  │   └─ policy_v2_suggestions.md             # 사람 피드백 기반 v2 정책 초안
  └─ notebooks/
      ├─ 1_build_comments_from_raw.ipynb
      ├─ 2_policy_simulation_comments_schema.ipynb
      ├─ 3_analysis_comments_results.ipynb
      └─ 4_policy_refinement_v1.ipynb
```

데이터 흐름:

1. **게시판 원시 텍스트 → `comments.csv`**  
   → `1_build_comments_from_raw.ipynb`
2. **A/B/C 정책을 이용한 LLM 시뮬레이션 → `comments_with_policy_results.csv`**  
   → `2_policy_simulation_comments_schema.ipynb`
3. **정책별 결과 분석 & 불일치 케이스 추출**  
   → `3_analysis_comments_results.ipynb`
4. **사람 피드백 + 정책 설명 → v2 정책 초안 (`policy_v2_suggestions.md`)**  
   → `4_policy_refinement_v1.ipynb`
5. (선택) v2 정책을 다시 2번 노트북에 반영해 재시뮬레이션 → **정책 결정의 iterative loop**

---

## 0. 환경 설정

### Python & 패키지

필요한 주요 패키지:

- `pandas`
- `tqdm`
- `python-dotenv`
- `openai`
- `matplotlib`
- (Jupyter) `notebook` or `jupyterlab`

예시:

```bash
pip install pandas tqdm python-dotenv openai matplotlib jupyter
```

### OpenAI API 키 설정

프로젝트 루트(`project_root/`)에 `.env` 파일을 만들고:

```env
OPENAI_API_KEY=sk-...
```

로 설정합니다. 모든 노트북은 `.env`에서 이 키를 읽어 LLM을 호출합니다.

---

## 1. 게시판 원시 텍스트 → `comments.csv`

**노트북:** `1_build_comments_from_raw.ipynb`

### 목적

게시판에서 긁어온 **원시 텍스트 덤프**를,  
LLM 시뮬레이션에 쓰기 좋은 **구조화된 CSV(`comments.csv`)**로 바꿉니다.

### 입력: `data/raw_everytime.txt`

예상 형식(대학 게시판 예시):

- 여러 개의 게시글이 하나의 텍스트 파일에 연속으로 들어 있음
- 각 게시글 블록은 특정 헤더(예: `게시판` 혹은 사이트에서 복사될 때 반복되는 구분 줄)를 기준으로 구분
- 한 게시글 블록 내부에는 대략:
  - 작성자 닉네임 (예: `익명`)
  - 날짜/시간 줄 (예: `11/26 23:55쪽지신고`)
  - 게시글 본문 여러 줄
  - 공감/스크랩 관련 숫자와 텍스트 (`020`, `공감`, `스크랩` 등)
  - 댓글 헤더 줄 (예: `익명1대댓글공감쪽지신고`, `닉네임대댓글공감쪽지신고`)
  - 댓글 본문 한 줄
  - 댓글 날짜/시간 줄 (`11/27 10:44`)  
  - 위 패턴이 반복

이 노트북은 이런 패턴을 이용해 **원글 / 댓글 / 대댓글**을 분리해 `comments.csv`로 저장합니다.

### 출력: `data/comments.csv`

스키마:

- `sample_id` : 전체에서 유니크한 ID (1, 2, 3, …)
- `thread_id` : 동일 게시글(스레드) ID
- `role` : `"post"` (원글) 또는 `"comment"` (댓글/대댓글)
- `order_in_thread` : 스레드 내 순서 (0=원글, 1부터 댓글)
- `text` : 텍스트 본문 (줄바꿈은 공백으로 합쳐짐)

---

## 2. A/B/C 정책 시뮬레이션 (LLM moderation)

**노트북:** `2_policy_simulation_comments_schema.ipynb`

### 목적

`comments.csv` 안에 있는 **각 문장(원글 + 댓글)** 에 대해:

- **Policy A / Policy B / Policy C** 라는 세 가지 규제 철학을 LLM에게 설명하고,
- 각 정책 하에서 이 댓글을
  - `BLOCK`
  - `WARN_AND_ALLOW`
  - `ALLOW`
  중 무엇으로 판단하는지,
- 그리고 `tone_score`, `debate_value_score`까지 같이 점수화하도록 합니다.

이를 통해 **정책별로 어떤 댓글이 막히고, 어떤 댓글이 그냥 지나가는지** 시뮬레이션할 수 있습니다.

### A/B/C 정책 개요

정책 텍스트는 프롬프트에 직접 삽입되며, 취지는 대략 다음과 같습니다.

- **Policy A: High Protection, Low Tolerance (매너 최우선)**
  - 인신공격, 조롱, 욕설, 혐오 표현, 위협 등은 매우 낮은 기준으로도 **BLOCK**
  - 시스템/정책 비판이라도 표현이 거칠면 `WARN_AND_ALLOW`
  - 애매하면 **“정서적 안전”을 우선 → 더 많이 차단**

- **Policy B: Balanced (매너와 토론의 균형)**
  - 직접적인 개인 공격, 혐오 표현, 폭력/도킹 위협 → **BLOCK**
  - 아이디어/정책/수업에 대한 거친 비판은 **경고+허용(`WARN_AND_ALLOW`)** 위주
  - 토론 가치가 있으면 가능한 한 살리되, 명백한 혐오는 허용하지 않음

- **Policy C: Minimal Regulation, High Freedom (최소 규제, 표현의 자유 우선)**
  - 심각한 인신공격+혐오+폭력 위협만 **BLOCK**
  - 대부분의 거친 언어는 `WARN_AND_ALLOW` 또는 `ALLOW`
  - 애매하면 **“표현의 자유”를 우선 → 최소 규제**

이렇게 정책 철학을 분리해 두면,
**“규제 강도에 따라 게시판 풍경이 얼마나 달라지는가?”**를 LLM을 통해 실험할 수 있습니다.

### LLM 프롬프트 구조

각 댓글 `text`에 대해 프롬프트를 구성:

- A/B/C 정책 설명을 붙이고
- 모델에게 요청:
  - `decision` ∈ {`BLOCK`, `WARN_AND_ALLOW`, `ALLOW`}
  - `short_reason` : 1–2문장 설명
  - `tone_score` : 1–5 (1=매우 공손, 5=매우 공격적)
  - `debate_value_score` : 1–5 (1=토론/정보 기여 거의 없음, 5=높음)
- 응답 형식은 **JSON**만 허용

결과를 파싱해서 행 단위로 저장합니다.

### 출력: `results/comments_with_policy_results.csv`

스키마 (1개의 원글/댓글 → 3개의 row):

- `sample_id`, `thread_id`, `role`, `order_in_thread`
- `policy` : `"A"`, `"B"`, `"C"`
- `decision`
- `short_reason`
- `tone_score`
- `debate_value_score`

---

## 3. 정책별 결과 분석

**노트북:** `3_analysis_comments_results.ipynb`

### 목적

- 정책별로 **BLOCK / WARN_AND_ALLOW / ALLOW 비율**을 비교하고,
- **tone_score / debate_value_score 평균값**을 비교하며,
- **A/B/C가 서로 다른 판단을 내린 사례**를 모아 봅니다.

이를 통해:

- “Policy A로 가면 어느 정도가 차단되는가?”
- “Policy C는 얼마나 느슨한가?”
- “특정 댓글에 대해 A는 BLOCK인데 C는 ALLOW인 사례가 얼마나 되는가?”

같은 질문에 답할 수 있습니다.

### 주요 분석 단계

1. `comments.csv` + `comments_with_policy_results.csv` 로드 후 merge
2. **정책별 결정 분포**:
   - `groupby(["policy", "decision"])` → 카운트/퍼센트
   - matplotlib로 간단한 막대 그래프
3. **정책별 평균 점수**:
   - `groupby("policy")[["tone_score", "debate_value_score"]].mean()`
   - 정책별로 “얼마나 공격적인 언어를 허용하고 있는지”, “토론 가치가 높은 댓글은 잘 살리고 있는지” 확인
4. **정책 간 불일치 케이스 추출**:
   - `pivot_table`로 `sample_id` 별 `A/B/C` 결정 한 줄에 모으기
   - `nunique(axis=1) > 1` 인 경우만 필터링
   - 원래 `text`까지 같이 붙인 뒤
   - `results/disagreement_cases_with_text.csv` 로 저장

불일치 케이스는 **“정책의 성격 차이가 드러나는 흥미로운 사례 모음”**으로,  
다음 단계인 **사람 피드백 + 정책 개선** 단계의 핵심 재료가 됩니다.

---

## 4. 사람 피드백을 이용한 정책 v2 초안 만들기

**노트북:** `4_policy_refinement_v1.ipynb`

### 목적

여기서부터가 이 프로젝트의 **핵심 실험 포인트**입니다.

> AI가 제안한 정책(A/B/C)과 시뮬레이션 결과를 보고,  
> 사람이 개입해서 “이 경우엔 이렇게 처리하는 게 더 낫다”라고 피드백을 주고,  
> **그 사람 피드백을 다시 LLM에 입력해서 정책 v2를 제안받는 과정**입니다.

이 과정을 반복하면,  
**정책 결정·수정·재적용의 iterative loop**를 AI와 사람이 함께 수행할 수 있습니다.

### 4-1. 사람 피드백 템플릿 생성

입력: `results/disagreement_cases_with_text.csv`  
출력: `results/disagreement_with_human_template.csv`

추가되는 컬럼:

- `human_preferred_decision`
  - 사람이 보기에 **가장 적절한** 판단: `BLOCK`, `WARN_AND_ALLOW`, `ALLOW`
- `human_comment` (선택)
  - 이유 설명:  
    - 예: `"자기비하라서 허용"`, `"성적 표현이지만 특정 대상 공격은 아니라 경고+허용"` 등

사람은 이 템플릿을 엑셀/VSCode 등으로 열어 **직접 라벨링**합니다.

라벨링이 끝나면, 예를 들어:

- `results/disagreement_with_human_filled.csv`

라는 파일 이름으로 저장합니다.

### 4-2. 사람 피드백 + 정책 v1 설명 → v2 정책 제안

노트북에서는:

1. `disagreement_with_human_filled.csv`를 로드
   - `human_preferred_decision`이 채워진 row만 사용
2. 기존 **Policy A/B/C v1 텍스트**를 문자열로 제공
3. 대표적인 피드백 케이스들을 텍스트 블록으로 정리
4. LLM에게 다음을 요청:

   - 현재 Policy A/B/C v1 설명과
   - 사람 피드백이 일치하지 않는 부분을 고려하여
   - **Policy A/B/C v2 설명을 제안**하라.

   조건:
   - A/B/C의 철학(강/중/약 규제)은 유지
   - BLOCK / WARN_AND_ALLOW / ALLOW 기준이 더 명료해지도록
   - 사람이 한 실제 선택과 더 잘 맞도록 조정
   - 각 정책별로:
     - 3–6개 원칙 bullet
     - BLOCK/WARN/ALLOW 기준
     - 2–3개의 예시 패턴(막아야 할 vs 허용해도 될 표현)을 포함

5. LLM 응답을 그대로 `results/policy_v2_suggestions.md`에 저장

이 파일은 **사람+AI가 공동으로 만든 v2 정책 초안**에 해당합니다.

---

## 5. Iteration: 정책 결정 과정의 순환 구조

이 프로젝트의 중요한 점은, 여기서 끝이 아니라는 것입니다.

1. **초기 정책 v1**은 사람이 대략 설계한 규칙입니다.
2. 이를 LLM 프롬프트로 넣어 게시판 데이터를 돌려 보면:
   - A/B/C 정책이 실제 댓글을 어떻게 처리하는지
   - 서로 얼마나 다르게 동작하는지
   - 어떤 케이스에서 사람이 보기엔 “과잉 차단/과도한 허용”이 일어나는지
   를 알 수 있습니다.
3. 이 **불일치·문제 케이스**에 대해 사람이 직접:
   - “여기는 실제로는 허용해야 한다”
   - “여기는 경고 정도가 적당하다”
   - “이 정도면 차단해야 한다”
   라고 **human_preferred_decision**을 남깁니다.
4. 그 사람 피드백을 다시 LLM에 입력해서
   - “이런 사례들에서 사람은 이렇게 판단했으니,
     기존 정책 문구를 어떻게 바꾸면 좋을까?” 를 묻습니다.
5. 이렇게 생성된 **Policy v2**를 다시 2번 노트북(시뮬레이션)에 반영해 돌리면,
   - 정책이 어떻게 변화했는지,
   - 댓글 규제 풍경이 어떻게 달라졌는지,
   - 사람 피드백과의 불일치가 줄어들었는지를 계속 확인할 수 있습니다.

이 전체 과정은:

> **“정책 결정 → 적용 → 문제 케이스 수집 → 사람 판단 → 정책 수정 → 재적용”**  

이라는, 원래 민주적·공론장 설계에서 중요한 **피드백 루프**를  
**AI와 LLM을 활용해서 보다 빠르게 실험할 수 있는 구조**로 만들려는 시도입니다.

---

## 6. 요약

이 프로젝트는 단순히 “AI로 댓글을 자동 차단해 보자”가 아닙니다.

- **A/B/C라는 서로 다른 규제 철학을 가진 정책을 정의**하고,
- 그 정책을 **AI에게 설명해 시뮬레이션하도록 한 뒤**,
- 사람이 개입해서 **정책의 문제점을 지적하고 수정 방향을 제시**하고,
- 그 정보를 다시 AI에 넣어 **정책 v2를 생성**하고,
- 이 과정을 반복하면서 **정책 결정·수정의 순환(iteration)을 실험**합니다.

즉,  
**“댓글 규제에 AI를 어떻게 사용할 수 있는가?”**라는 질문에 대해,

> - AI = **정책 집행자**가 아니라,  
> - AI = **정책 실험과 학습을 도와주는 도구**  

로 쓸 수 있다는 것을 보여주는 작은 프로토타입입니다.
