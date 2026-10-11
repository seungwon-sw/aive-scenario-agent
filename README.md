# AIVE — 밈 활용 숏폼 광고 영상 자동 생성 파이프라인

> 전체 프로젝트 레포지토리: [SKN19-Final-4team](https://github.com/SKN19-Final-4team) (비공개)<br>
> 본 문서는 프로젝트 전체가 아닌, **시나리오 생성 에이전트 및 콘텐츠 파이프라인**에 대한 본인 기여 부분을 정리한 것입니다.


## 프로젝트 개요

**AIVE**는 광고주가 제품 정보를 입력하면, 트렌드 밈을 활용한 숏폼 광고 영상을 시나리오 기획부터 영상 합성까지 자동으로 생성하는 멀티 에이전트 파이프라인입니다.

밈 수집 에이전트가 웹에서 밈을 수집·분석하고, 시나리오 에이전트가 제품과 밈을 결합해 4씬 구조의 광고 시나리오를 생성하면, TTS·영상 생성 노드가 이를 소비해 최종 영상을 합성합니다.

기간: 2025-12-18 ~ 2026-02-11 · 팀 5명


### 담당 범위

| 단계 | 파일 | 설명 |
|------|------|------|
| 시나리오 스키마 설계 | `scenario/scenario_schemas.py` | 3단 구조(Hook/Body/Close)의 structured output 스키마, 검수 결과 스키마 |
| 에이전트 그래프 구성 | `scenario/scenario_graph.py` | 생성 → 검수 → 재생성 조건부 루프, 재시도 상한 처리 |
| 노드 구현 | `scenario/scenario_nodes.py` | 생성·검수·재생성·저장 4개 노드 |
| 프롬프트 형상관리 | `scenario/prompts/`, `scenario/scenario_database.py` | 프롬프트 버전 분리 및 DB 적재, 생성 이력 추적 |
| 밈 분석 프롬프트 | `meme_collector/prompts/` | 활용 예시 추출, 리스크 카테고리 분류 |
| 밈 추천 정렬 | `meme_collector/meme_sorting.py`, `embed_situations.py` | 제품 설명–밈 상황 임베딩 유사도 기반 정렬 |
| BGM 생성 노드 | `content_pipeline/voice/` | ElevenLabs Music 연동, 생성 프롬프트 설계 *(최종 미채택)* |
| 영상 생성 서버리스 전환 | `content_pipeline/video/service.py` | ComfyUI SSH → RunPod Serverless 전환 |


### 기술 스택

| 분야 | 사용 도구 |
|------|-----------|
| **Language** | [![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=Python&logoColor=white)](https://www.python.org/) |
| **Collaboration Tool** | [![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)](https://git-scm.com/) [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/) [![Notion](https://img.shields.io/badge/Notion-000000?style=for-the-badge&logo=notion&logoColor=white)](https://www.notion.so/) |
| **LLM Model** | [![GPT-4o](https://img.shields.io/badge/GPT--4o-412991?style=for-the-badge&logo=openai&logoColor=white)](https://platform.openai.com/) [![Gemini](https://img.shields.io/badge/Gemini-8E75B2?style=for-the-badge&logo=googlegemini&logoColor=white)](https://ai.google.dev/) |
| **Embedding Model** | [![BGE-m3-ko](https://img.shields.io/badge/dragonkue%2FBGE--m3--ko-FFD21E?style=for-the-badge&logo=huggingface&logoColor=black)](https://huggingface.co/dragonkue/BGE-m3-ko) [![sentence-transformers](https://img.shields.io/badge/sentence--transformers-FFD21E?style=for-the-badge&logo=huggingface&logoColor=black)](https://www.sbert.net/) |
| **Orchestration** | [![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=for-the-badge&logo=langchain&logoColor=white)](https://www.langchain.com/) [![LangGraph](https://img.shields.io/badge/LangGraph-000000?style=for-the-badge)](https://langchain-ai.github.io/langgraph/) |
| **Database** | [![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/) |
| **Generative Model** | [![ElevenLabs](https://img.shields.io/badge/ElevenLabs-000000?style=for-the-badge&logo=elevenlabs&logoColor=white)](https://elevenlabs.io/) [![ComfyUI](https://img.shields.io/badge/ComfyUI-1A1A1A?style=for-the-badge)](https://github.com/comfyanonymous/ComfyUI) |
| **Infra** | [![RunPod](https://img.shields.io/badge/RunPod%20Serverless-673AB7?style=for-the-badge)](https://www.runpod.io/) [![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/) [![AWS S3](https://img.shields.io/badge/AWS%20S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)](https://aws.amazon.com/s3/) |
| **Development Env** | [![VS Code](https://img.shields.io/badge/VS%20Code-007ACC?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://code.visualstudio.com/) [![uv](https://img.shields.io/badge/uv-DE5FE9?style=for-the-badge&logo=uv&logoColor=white)](https://docs.astral.sh/uv/) |

<br>

---

## 1. 시나리오 스키마 설계 — 후속 노드가 소비 가능한 출력 만들기

### 문제

이 노드의 출력은 사람이 읽는 문서가 아니라 TTS·영상 생성 노드가 소비하는 입력입니다. 후속 파이프라인이 씬 단위로 동작하기 때문에, LLM이 자유 형식으로 시나리오를 작성하면 씬 개수와 필드 구성이 매번 달라져 파싱이 실패하고 영상 길이도 들쭉날쭉해졌습니다.

즉 요구사항은 "좋은 시나리오"가 아니라 "구조가 고정된 시나리오" 였습니다.

### 3단 구조 채택 근거

구조를 임의로 정하지 않고, 광고 업계에서 사용되는 **Hook–Body–CTA** 프레임워크를 조사해 채택했습니다.

| 구간 | 씬 | 길이 | 역할 |
|------|-----|------|------|
| **Hook** | scene1 | 3~5초 | 시선 즉시 포착 — 핵심 가치 제시 / 문제 상황 / pattern interrupt |
| **Body** | scene2, scene3 | 각 7~9초 | 브랜드 친밀감, 회상률, 제품 가치 전달 |
| **Close** | scene4 | 3~5초 | 행동 유도 문구(CTA) |

숏폼은 초반 3초 이탈률이 지배적이므로 Hook에서는 서사 전개보다 주목 확보를 우선했고, **밈을 Hook에 고정 배치**했습니다. 밈 자체가 pattern interrupt 역할을 하기 때문입니다.

씬별 출력 필드는 **후속 노드가 무엇을 필요로 하는가**에서 역산해 3개로 확정했습니다.

| 필드 | 내용 | 소비 노드 |
|------|------|-----------|
| `dialogue` | `(감정) 대사` 형태 | TTS — 괄호 안 감정 태그로 발화 톤 제어 |
| `action` | 캐릭터 행동·표정 | 영상 생성 — 모션 프롬프트 |
| `visual_description` | 배경·소품·화면 구성 | 영상 생성 — 장면 프롬프트 |

감정을 별도 필드로 분리하지 않고 대사에 인라인으로 넣은 이유는, 한 씬 안에서 톤이 바뀌는 경우를 표현하기 위해서입니다.

### 시행착오 과정

**1차: `Dict[str, Scene]` 동적 스키마**

씬 개수를 유연하게 두기 위해 `scenes: Dict[str, Scene]` 형태로 설계했습니다. 그러나 LLM structured output이 동적 Dict 스키마를 안정적으로 반환하지 못했습니다. 키 이름이 매번 달라지거나 아예 스키마 검증에 실패하는 경우가 반복됐습니다.

**2차 (최종): `scene1`~`scene4` 고정 필드로 평탄화**

유연성을 포기하고 4개 고정 필드로 전환했습니다. 애초에 3단 구조를 강제하는 것이 목적이었으므로 유연성은 요구사항이 아니었습니다. 스키마 설계 전에 structured output의 제약을 먼저 검증했어야 했다는 것이 이 전환에서 얻은 교훈입니다.

**3차: 글자 수 제약의 위치 변경**

`max_length`로 대사 길이를 강제했더니 문장이 중간에 잘리는 결과가 나왔습니다. 스키마 검증은 통과/실패만 있어서 "가급적 짧게"라는 의도를 표현할 수 없기 때문입니다. **하드 제약(스키마) → 소프트 유도(프롬프트)** 로 옮기고, 프롬프트에서 글자 수 지시사항을 강화하는 방향으로 전환했습니다.

### 최종 스키마

```python
class StructureType(str, Enum):
    HOOK  = "hook"   # 3-5초, 시선 포착
    BODY  = "body"   # 7-9초, 밈 활용 스토리
    CLOSE = "close"  # 3-5초, CTA

class Scene(BaseModel):
    scene_type: StructureType
    dialogue: str            # (감정) 대사 — TTS용
    action: str              # 캐릭터 행동·표정 — 영상 생성용
    visual_description: str  # 배경·소품·화면 구성 — 영상 생성용

class Scenario(BaseModel):
    scene1: Scene   # Hook
    scene2: Scene   # Body 1
    scene3: Scene   # Body 2
    scene4: Scene   # Close
    title: str          = Field(..., max_length=50)
    description: str    = Field(..., max_length=200)
    hashtags: list[str] = Field(..., min_length=3, max_length=5)
```

**참고 자료**
- [Hook–Body–CTA Video Editor Guide — Sovran.ai](https://sovran.ai/blog/hook-body-cta-video-editor-the-ultimate-guide-to-creating-high-converting-video-ads-in-2025)
- [Creative Best Practices: Top Performing Ads — TikTok for Business](https://ads.tiktok.com/business/en/blog/creative-best-practices-top-performing-ads)

---

## 2. 검수·재생성 루프 — 정성 평가를 정량 축으로 분해

### 문제

생성된 시나리오를 그대로 영상으로 넘기면 품질 편차가 컸습니다. 검수 노드를 붙이기로 했으나, 검수 결과가 "좋다/나쁘다"로만 나오면 **재생성 프롬프트에 넣을 피드백이 만들어지지 않는다**는 문제가 있었습니다.

### 설계: 5개 축 × 4개 씬

평가를 축으로 분해하고, 씬별로 점수를 매기도록 스키마를 설계했습니다.

```python
class SceneReview(BaseModel):
    scene_key: str = Field(..., pattern="^scene[1-4]$")
    brand_safety_score: float           # 브랜드 안전성 (혐오·차별·리스크)
    message_delivery_score: float       # 메시지 전달력
    character_consistency_score: float  # 캐릭터 일관성
    meme_usage_score: float             # 밈 활용 적절성
    completeness_score: float           # 완성도
    feedback_items: list[FeedbackItem]  # 축별 구체적 개선사항

class ReviewResult(BaseModel):
    scene_reviews: list[SceneReview] = Field(..., min_length=4, max_length=4)
    total_score: float = Field(..., ge=0, le=200)  # 5축 × 10점 × 4씬
    approved: bool
```

- **씬별로 쪼갠 이유**: 전체 점수만 있으면 어느 씬을 고쳐야 할지 알 수 없습니다. 씬별 점수가 있어야 재생성 시 문제가 된 씬을 지목할 수 있습니다.
- **`feedback_items`의 역할**: 점수는 승인/재생성 판정용이고, `description`은 재생성 프롬프트(`REGENERATE_PROMPT_V1`)에 그대로 주입됩니다.

### 그래프 구성

```
START → generate_scenario → review_scenario ─┬─ (승인 또는 재시도 3회 초과) ─→ save_scenario → END
                                 ↑           │
                                 │           └─ (불승인 또는 휴먼 피드백) ─→ regenerate_scenario
                                 └─────────────────────────────────────────────┘
```

```python
def should_regenerate(state: ScenarioState) -> str:
    review = state["review_result"]

    # 휴먼 피드백은 점수와 무관하게 무조건 재생성
    if review["feedback_type"] == "human_feedback":
        return "regenerate"

    if review["feedback_type"] == "llm_review":
        if review["approved"]:
            state["status"] = "completed"
            return "end"
        if state.get("retry_count", 0) >= 3:      # 무한 루프 방지
            state["status"] = "failed"
            return "end"
        return "regenerate"

    return "end"
```

### 판단 근거

| 결정 | 이유 |
|------|------|
| 휴먼 피드백과 LLM 검수를 하나의 노드에서 분기 | 그래프를 분리하면 재생성 프롬프트가 이원화됨. `feedback_type`으로만 갈라 단일 경로 유지 |
| 재시도 상한 3회 | 승인되지 않는 시나리오가 무한 루프를 도는 것을 방지 |
| 상한 초과 시 예외가 아닌 `status="failed"` | 광고 한 건 때문에 파이프라인 전체가 멈추면 안 됨 |
| `save_scenario`를 성공·실패 공통 경로로 | 실패한 시나리오도 남아야 어떤 프롬프트에서 실패했는지 분석 가능 |

---

## 3. 프롬프트 형상관리 — 생성 결과의 재현 가능성 확보

### 문제

프롬프트를 수정하면서 시나리오 품질이 나아지는지 판단할 근거가 없었습니다. 프롬프트가 코드 안에 문자열로 박혀 있어 버전 간 비교가 불가능했고, 생성된 결과물만 봐서는 어떤 프롬프트로 만들어진 것인지 알 수 없었습니다.

### 설계: 프롬프트 분리 → 버전 테이블 → 생성 이력 기록

**1) 프롬프트를 코드에서 분리하고 파일로 관리**

```
scenario/prompts/
├── generate_scenario_system_templates.py   # system 프롬프트
├── generate_scenario_user_templates.py     # user 프롬프트
├── regenerate_scenario_templates.py        # 재생성
├── review_scenario_templates.py            # 검수
└── meme_example_templates.py               # 밈 활용 예시 주입
```

system/user를 분리한 것은 파인튜닝 데이터셋 변환을 염두에 둔 것입니다. 하나의 문자열로 합쳐져 있으면 학습 포맷으로 변환할 수 없습니다.

**2) `prompt_versions` 테이블 — 프롬프트 자체를 버전 관리**

```python
# 템플릿에서 {변수} 를 정규식으로 자동 추출해 함께 저장
variables = re.findall(r'\{(\w+)\}', template)

INSERT INTO prompt_versions (prompt_name, version, content, variables, is_active)
```

변수를 자동 추출해 저장하면, 템플릿이 요구하는 변수와 실제 전달 변수의 불일치를 검증할 수 있습니다. 동일 `prompt_name` + `version`이 이미 있으면 건너뛰어 중복 적재를 막았습니다.

**3) `scenario_scripts` 테이블 — 생성 이력 전체 기록**

```python
INSERT INTO scenario_scripts (
    ad_id, meme_id, scenes,
    used_model,        # 사용 모델
    generation_type,   # initial / llm_review_v1 / llm_review_v2 / human_v1
    used_templates,    # 사용한 템플릿 조합
    system_prompt,     # 실제 투입된 system 프롬프트
    user_prompt,       # 실제 투입된 user 프롬프트
    full_response,     # 모델 응답 원문
    thinking           # CoT 부분
)
# 이후 update_review_result_to_db() 로 같은 레코드에 검수 결과 갱신
```

### 결과

생성된 시나리오 한 건을 집어서 **어떤 프롬프트 버전 + 어떤 모델 + 어떤 입력으로 생성되었고, 검수 점수는 몇 점이었는지**를 역추적할 수 있게 되었습니다.

`thinking`(CoT)까지 남긴 것은 이후 파인튜닝 데이터로 전환할 수 있도록 한 것입니다.


---

## 4. 밈 추천 정렬 — 제품 설명과 밈 상황의 임베딩 유사도

### 문제

광고주가 제품 정보를 입력하면 어떤 밈을 쓸지 골라야 하는데, 초기에는 밈 목록을 단순 나열해 사용자가 직접 고르는 방식이었습니다. 제품과 무관한 밈이 상단에 노출되는 문제가 있었습니다.

### 설계

밈 데이터에는 밈이 사용되는 **상황(`situation`)** 텍스트가 있습니다. 제품 설명과 이 상황 텍스트를 임베딩해 코사인 유사도로 정렬했습니다.

```python
MODEL_NAME = "dragonkue/BGE-m3-ko"

def sort_memes_by_similarity(item_description: str, top_n: int = 10) -> list[dict]:
    item_emb = _get_model().encode([item_description], normalize_embeddings=True)[0]
    # DB에 사전 계산된 situation_embedding 과 dot product (normalize 되어 있음)
```

### 실험과 판단 근거

| 실험 | 비교 대상 | 결정 |
|------|-----------|------|
| 임베딩 대상 텍스트 | `situation` 단독 vs `definition + situation + dialogue_example` | **`situation` 단독 채택** |
| 임베딩 모델 | `jhgan/ko-sroberta-multitask` vs `dragonkue/BGE-m3-ko` | **BGE-m3-ko 채택** |
| 임베딩 계산 시점 | 요청 시점 계산 vs 사전 계산 후 DB 저장 | **사전 계산** — 요청 시점 임베딩은 지연이 큼 |
| 모델 로딩 시점 | 요청마다 로딩 vs 서버 시작 시 로딩 | **서버 시작 시 로딩** — 요청당 모델 로딩 오버헤드 제거 |

**임베딩 대상 텍스트를 `situation` 단독으로 결정한 이유**

정보를 더 넣을수록 좋을 것이라 예상하고 `definition + situation + dialogue_example`을 결합해봤으나, 오히려 결과가 나빠졌습니다. `definition`과 `dialogue_example`에는 "마케팅에 활용", "신곡 홍보에 활용" 같은 **밈의 사용처를 설명하는 메타 서술**이 섞여 있어, 제품 내용이 아니라 "홍보·마케팅"이라는 공통 어휘에 유사도가 끌렸습니다.

| 제품 설명 | 결합 방식 상위 결과 | `situation` 단독 상위 결과 |
|---|---|---|
| 초경량 무선 이어폰,<br>운동할 때 안 빠지는 | 골반통신 — *마케팅에 활용*<br>골반이 안 멈추는데 — *신곡 홍보에 활용* | **운동 많이 된다 —<br>운동선수들이 운동하는 모습** |

밈이 **어디에 쓰였는지**가 아니라 **어떤 상황을 담고 있는지**가 제품과 맞춰야 할 축이었습니다. 결합 텍스트는 그 축을 흐리기만 했습니다.

다만 채택 후에도 상위 유사도가 0.35 내외에 머물렀습니다. 제품 설명과 밈 상황은 어휘 공간이 달라 절대값이 낮게 나오므로, 이 점수는 **순위 결정에만 사용하고 임계값으로 잘라내지는 않도록** 설계했습니다.

`situation_embedding` 컬럼을 추가하는 마이그레이션(`002_add_situation_embedding.sql`)을 작성하고, `embed_situations.py`로 전체 백필 및 밈 단위 갱신(`embed_by_meme_id`)을 지원했습니다.

이 기능은 AI 레포의 정렬 로직뿐 아니라 백엔드 API(`sort_memes`)와 프론트엔드 연동까지 직접 작업했습니다.

---

## 5. 영상 생성 — ComfyUI SSH → RunPod Serverless 전환

### 문제

영상 생성은 상시 켜둔 GPU 서버에 SSH로 접속해 ComfyUI를 실행하는 구조였습니다. 사용하지 않는 시간에도 GPU 비용이 발생했습니다.

### 설계 판단

**기존 SSH 경로를 삭제하지 않고 분기로 유지**

```python
if self.config.mode == "serverless":
    ...
```

서버리스가 불안정할 경우 즉시 되돌릴 수 있도록 두 경로를 공존시켰습니다. 프로젝트 막바지(발표 3일 전) 작업이었기 때문에 롤백 가능성을 확보하는 것이 우선이었습니다.

**파일 전송 방식 재설계**

서버리스 워커는 로컬과 파일시스템을 공유하지 않습니다. SSH 방식에서 쓰던 파일 경로 전달이 불가능해, 참조 이미지·오디오를 **base64로 인코딩해 요청 본문에 싣고, 결과 영상을 디코딩**하는 방식으로 전면 재설계했습니다.

```
_encode_file_base64_serverless()      # 입력 파일 인코딩
_upload_reference_image_serverless()  # 참조 이미지 전달
_upload_reference_audio_serverless()  # 참조 오디오 전달
_submit_prompt_serverless()           # 워크플로우 제출
_poll_status_serverless()             # 비동기 상태 폴링
_download_output_serverless()         # 결과 디코딩·저장
```

씬별 영상 병합도 원격 실행이 불가능해 `_local_merge_scene_videos()`를 별도로 구현했습니다.

### 서버리스 워커 이미지

RunPod Serverless에서 구동할 ComfyUI 워커 이미지는 별도 저장소로 관리했습니다.

**[seungwon-sw/my_comfyui_ltx](https://github.com/seungwon-sw/my_comfyui_ltx)**

| 파일 | 내용 |
|------|------|
| `Dockerfile` | `runpod/worker-comfyui` 기반 이미지 빌드 |
| `example-request.json` | LTX-2 이미지+오디오→영상 워크플로우 및 API 요청 페이로드 |

**버전 호환성 문제와 커밋 고정**

이미지 초안은 ComfyUI 본체와 커스텀 노드를 최신 버전으로 받아오는 구성이었습니다. 이 상태로 빌드했을 때 팀이 로컬에서 쓰던 워크플로우가 그대로 동작하지 않는 문제가 있었습니다.

원인은 버전 불일치였습니다. 커스텀 노드는 업스트림 변경이 잦아 최신으로 받아오면 로컬 환경과 노드 스펙이 어긋납니다. 팀에서 실제로 사용 중인 버전을 확인한 뒤, ComfyUI 본체와 커스텀 노드 4종(KJNodes, VideoHelperSuite, Custom-Scripts, LTXVideo)을 모두 **해당 커밋에 고정**했습니다.

서버리스는 요청마다 컨테이너가 새로 뜨기 때문에, 이미지가 고정되어 있지 않으면 같은 워크플로우가 실행 시점에 따라 다르게 동작합니다. 로컬에서 검증한 환경과 워커 환경을 일치시키는 것이 전제 조건이었습니다.

모델(LTX-2 19B distilled fp8, Gemma-3 12B 텍스트 인코더, LoRA 2종)은 빌드 타임에 이미지에 포함시켜, 콜드스타트마다 수 GB를 내려받지 않도록 했습니다.

---

## 6. 전체 아키텍처 요약

```
[광고주 입력: 제품 정보]
        │
        ▼
[밈 추천 정렬]  ── 제품 설명 ↔ 밈 situation 임베딩 유사도 (본인 담당)
        │
        ▼
[시나리오 에이전트]  ── LangGraph (본인 담당)
        │
        ├── generate_scenario   ── 3단 구조 structured output
        ├── review_scenario     ── 5축 × 4씬 정량 검수
        ├── regenerate_scenario ── 피드백 주입 후 재생성 (최대 3회)
        └── save_scenario       ── 프롬프트·응답·검수결과 DB 적재
        │
        ▼
[콘텐츠 생성]
        ├── TTS (Qwen-TTS)          ── dialogue 소비
        ├── BGM (ElevenLabs Music)  ── 본인 담당(최종적으로는 BGM을 채택하지 않음)
        └── 영상 (ComfyUI)          ── action + visual_description 소비
        │
        ▼
[영상 합성]  ── RunPod Serverless / SSH 분기 (본인 담당)
```

### 데이터 흐름

```
[PostgreSQL]
   ├── ad_requests / companies / company_characters  ──→ 제품·캐릭터 정보 로드
   ├── memes / meme_examples                          ──→ 밈 정보 + 활용 예시 로드
   │      └── situation_embedding                     ──→ 밈 추천 정렬
   ├── prompt_versions                                ──→ 프롬프트 버전 관리
   └── scenario_scripts                               ──→ 생성 이력 + 검수 결과 적재
```

---

## 프로젝트 구조

```
AI/
├── content_pipeline/
│   ├── scenario/                            # 시나리오 에이전트 (본인 담당)
│   │   ├── scenario_graph.py                #   그래프 구성·조건부 분기
│   │   ├── scenario_nodes.py                #   생성·검수·재생성·저장 노드
│   │   ├── scenario_schemas.py              #   structured output 스키마
│   │   ├── scenario_state.py                #   노드 간 State 정의
│   │   ├── scenario_database.py             #   프롬프트·생성이력 DB 적재
│   │   └── prompts/                         #   프롬프트 템플릿 (버전별)
│   │
│   ├── voice/                               # TTS·BGM (BGM 노드 본인 담당, 최종 미채택)
│   ├── video/                               # 영상 생성 (서버리스 전환 본인 담당)
│   ├── image/                               # 이미지 생성 (타 팀원)
│   └── orchestrator.py                      # 전체 파이프라인 조율 (타 팀원)
│
├── meme_collector/                          # 밈 수집 에이전트
│   ├── meme_sorting.py                      #   밈 추천 정렬 (본인 담당)
│   ├── embed_situations.py                  #   situation 임베딩 백필 (본인 담당)
│   ├── prompts/                             #   분석·검증 프롬프트 (본인 담당)
│   ├── workers/, nodes/, tools/             #   수집·분석 워커 (공동)
│   └── deep_research/                       #   심층 조사 그래프 (공동)
│
├── finetune/                                # 시나리오 파인튜닝 (타 팀원)
├── common/                                  # 공통 설정·DB (타 팀원)
├── migrations/                              # DB 마이그레이션
└── tests/
```

관련 레포지토리 (본인 기여분)

| 레포 | 기여 |
|------|------|
| `AI` | 시나리오 에이전트, 밈 분석 프롬프트, 밈 정렬, BGM(미채택), 영상 서버리스 전환 |
| `data-pipeline` | YouTube 수집·Gemini 영상 분석, 발화자 정보 추출 |
| `content-generator` | 영상 생성 모델 PoC (D-ID / Pika / Hedra 비교) |
| `backend` | 밈 정렬 API, 임베딩 모델 로딩 시점 개선 |
| `frontend` | 밈 정렬 UI 연동 |
| `project-plan` | 저장소 구조 설계, 개발 규칙 문서화 |

---

## 회고

이 프로젝트에서는 한 주제를 깊이 파는 것과 전체 파이프라인을 동작시키는 것 중 후자를 우선하기로 선택했습니다. 부트캠프에서 인플루언서라는 대주제를 받았고, 처음에는 특정 인플루언서 하나를 만드는 방향으로 준비했습니다. 그러나 중간 평가에서 개별 인플루언서보다 시스템화가 낫겠다는 외부 평가자의 의견을 받았고, 이를 반영해 광고 제작 시스템으로 주제를 전환했습니다. 이에 따라 시나리오도 특정 페르소나의 등장인물을 고정하는 방식에서, 입력된 가상 회사에 맞는 광고 영상 시나리오를 생성하는 방식으로 바뀌었습니다. 시나리오의 완성도나 재미처럼 깊게 파볼 수 있는 주제보다는 운영적인 측면의 작업에 중점을 뒀습니다.

그래서 기능 구현 외에 프롬프트 형상관리에 신경을 썼습니다. 팀에서 프롬프트를 코드 안에 문자열로 두고 수정하는 동안, 프롬프트를 파일로 분리하고 버전을 나눈 뒤 버전별 수정사항을 정리했습니다. 이후에는 프롬프트 버전과 실행 결과를 DB에도 남겨 프롬프트 개선을 위한 기반을 만들어두려 했습니다. 로그를 쌓는 구조는 만들었지만, 그 데이터로 실제 의사결정을 내리는 단계까지는 가지 못했습니다. 프롬프트 선택 방식으로 ELO 레이팅 시스템을 계획해 코드로 구현하기도 했는데, 필요한 비교 횟수가 너무 많다는 한계 때문에 적용하지 못했습니다. 사람이 직접 비교해야 하는 횟수가 적은 방식으로 계획했어야 했다고 생각합니다. 시간이 더 있고 실제 사람이 사용하는 환경이었다면, 사용자에게 두 답변을 제공해 비교받거나 시나리오 초안의 피드백률을 통해 프롬프트를 수정하는 방식도 좋았을 것 같습니다.

(추후 검수 루프 확인 중 개선점이 생각나 남깁니다.)  
검수 루프에서도 아쉬운 점이 있습니다. 승인 기준은 총점 200점 중 160점 이상이고, 브랜드 안전성이 6점 미만인 씬이 하나라도 있으면 점수와 무관하게 불승인입니다. 그런데 이 합산과 판정을 검수 프롬프트 안에서 LLM이 하도록 두었고, 그래프는 LLM이 낸 approved 값으로만 분기했습니다. 분기는 코드로 고정했지만 분기의 입력은 LLM에 맡긴 셈입니다. 다시 만든다면 LLM은 항목별 점수와 피드백만 내게 하고, 합산과 기준 판정은 코드에서 한 뒤 조건부 엣지로 넘기겠습니다. llm 답변의 정확성이 흔들릴 때도 진행의 안정성은 조금은 더 보장할 수 있을 것이라고 생각하기 때문입니다. 

파이프라인 중간 단계를 맡다 보니, 앞뒤 노드가 동시에 개발되는 상황에서 제 출력이 어떻게 소비될지 확정되지 않은 채로 만들어야 했습니다. 그 결과 씬 구조와 출력 형식을 여러 번 바꾸게 되었습니다. 또한 뒷단의 영상 생성과 연결되는 시점이 늦어져, 생성된 영상을 보고 시나리오 에이전트를 수정하는 피드백 루프를 돌리지 못했습니다. 연결되는 모듈 간 인터페이스를 먼저 합의하고, 결과를 확인할 수 있는 최소한의 연결을 일찍 만드는 것이 중요하다고 느꼈습니다.

프로젝트 과정에서 제미나이 모델에 유튜브 URL을 주고 영상 분석을 받거나, BGM 모델을 프롬프팅해 사용해 보는 등(프롬프트로 음악을 안정적으로 뽑아내기 어려워 테스트 후 폐기) 기존에 학습하지 않은 새로운 방식들을 배울 수 있었습니다. 사용자가 어떤 입력을 줄지, 어떤 방식으로 밈 선택을 유도할지를 고민해본 것도 의미 있었습니다.
