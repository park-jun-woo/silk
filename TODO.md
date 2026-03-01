# SILK TODO

## 현황 요약

GEUL 프로젝트에서 위키데이터 인코딩 작업 상당 부분 완료:
- **geuldev**: 위키데이터 117.4M 엔티티, 17억 트리플 적재 완료
- **geulwork**: 108.8M 엔티티 SIDX 인코딩 완료, 63개 타입 정의, 142K 코드북
- **bit_allocation**: 5개 타입 속성 48비트 배치 완료 (Human, Star, Settlement, Organization, Film)
- **미분류**: 19.7M개 (entity_type=63, attrs=0)

SILK PoC는 geulwork의 인코딩 결과를 NumPy 인덱스로 변환하여 사용.
코드북 산출(타입 정의, 비트 할당, 인코딩, 검증)은 별도 프로젝트 **GEUL 하위 프로젝트** 범위.
SILK은 GEUL 하위 프로젝트가 생성한 data/ 산출물만 소비한다.

---

## Phase 구분

| Phase | 이름 | 섹션 | 성격 | 세션 |
|-------|------|------|------|------|
| **A** | Export + 코어 | 1, 2, 3 | 기계적. Claude Code 자율 진행 가능 | 1-2 |
| **B** | 논문 초안 | 5 | 전략적. 사용자 판단 필요 (대화형) | 1 |
| **C** | 데모 | 4, 6 | A 결과물로 동작 확인하며 진행 | 1-2 |
| **D** | 벤치마크 | 7 | 외부 의존 (faiss-cpu, 임베딩 모델) | 1-2 |
| **E** | 마무리 | 8 | D 결과 숫자로 논문 완성 + 배포 | 1 |

```
Phase A  →  Phase B (병렬 가능)
    └──────────┘
          ↓
       Phase C
          ↓
       Phase D
          ↓
       Phase E
```

---

# Phase A: Export + 코어

## 1. 코드북 데이터 (data/) `[A]`

**GEUL 하위 프로젝트** 프로젝트가 생성한 산출물을 data/에 배치. 모든 후속 작업의 전제조건.

SILK은 코드북을 만들지 않는다. GEUL 하위 프로젝트의 결과물을 가져다 쓴다.

### GEUL 하위 프로젝트 → data/ 연동
- [ ] GEUL 하위 프로젝트에서 산출물 복사 또는 심볼릭 링크
  - `data/entity_types.json` — 63개 타입, 카테고리, QID
  - `data/bit_allocation.json` — 5개 타입 38 필드, offset/width
  - `data/codebook.json` — 142K 엔트리, 타입별 필드→코드 매핑
  - `data/fips_to_qid.json` — FIPS 국가코드 → Q-ID
  - `data/cameo_to_wordnet.json` — CAMEO EventCode → 워드넷 synset
  - `data/verb_codebook.json` — 동사 코드북 (WordNet 기반)

### 라벨 데이터 (검색 결과 표시용)
- [ ] Q-ID → 영문 라벨 추출 (`scripts/export_labels.py`)
  - geuldev.entity_labels (en) → `data/qid_labels.json` 또는 경량 SQLite
  - 108.8M개 전체는 비현실적 → 주요 타입 또는 빈도순 상위 N개로 제한

### GEUL 하위 프로젝트 범위 (이 레포 밖)
- 타입 정의, 비트 할당, 코드북 빌드, 인코딩, VALID 검증
- 미분류 19.7M개 재분류 (entity_type=63)
- hierarchy 적재, P279 체인 탐색, 커버리지 개선

## 2. 코어 라이브러리 `[A]`

silk/ 패키지. GEUL 하위 프로젝트 산출물(data/)을 로드하여 검색 수행.

### 패키지 설정
- [ ] `silk/__init__.py` — 패키지 초기화
- [ ] `pyproject.toml` 의존성 확정 (numpy, psycopg2-binary 등)
- [ ] `pip install -e .` 로컬 설치 확인

### 코어 모듈 (5개)
- [ ] `silk/codebook.py` — data/ json 로드, 필드→코드 lookup, 코드→필드 역변환
- [ ] `silk/encoder.py` — JSON 태그 → SIDX uint64 비트 조립
- [ ] `silk/valid.py` — VALID 검증 (유효값, 타입 정합성, 제약조건)
- [ ] `silk/search.py` — NumPy 비트 AND 검색 (sidx_array + qid_array 로드)
- [ ] `silk/query.py` — 쿼리 dict → mask/pattern 생성

## 3. 위키데이터 인덱스 변환 `[A]`

108.8M 엔티티 → NumPy 인덱스. **인코딩은 완료. 변환만 남음.**

### 완료 (geulwork)
- [x] P31 기반 Q-ID → type 매핑 (63개 타입, 108.8M 인코딩 완료)
- [x] 속성 태깅 (5개 타입 bit_allocation + codebook 기반)
- [x] 비트 조립 → geulwork.entity_sidx (sidx + attrs, 9.5GB)

### 변환
- [ ] `scripts/build_index.py` — geulwork.entity_sidx → NumPy 변환
  - sidx (uint64) + qid_num (uint32) 병렬 배열
  - `index/entity_sidx.npy` (~870MB) + `index/entity_qid.npy` (~435MB)
- [ ] 인덱스 정렬 (sidx 순, qid 배열도 동기 정렬)
- [ ] VALID 검증 실행 — 인코딩 결과 정합성 체크
  - 코드북 유효값 범위 확인
  - 타입별 attrs 비트 범위 검증
  - 샘플링 spot-check (원본 triples와 대조)

---

# Phase B: 논문 초안

## 5. 논문 초안 `[B]`

데모·벤치마크 전에 구조를 먼저 잡는다. 필요한 숫자와 시나리오가 확정되어야 뭘 만들지 안다.

- [ ] 논문 골격 작성 (`paper/draft.md` 또는 LaTeX)
  - Section 구성, 각 섹션별 핵심 주장 1줄, 필요한 Figure/Table 목록
- [ ] Section 1: Introduction — RAG 한계, 검색은 구조 없는 데이터의 증상
- [ ] Section 2: SILK Architecture — 뉴로-심볼릭, 역할 분담
- [ ] Section 3: SIDX Encoding — 64비트 레이아웃, 코드북, 우아한 열화
- [ ] Section 4: VALID Pipeline — 3단계 검수, 환각 방어
- [ ] Section 5: Event SIDX — GDELT 변환, CAMEO→워드넷
- [ ] Section 6: Search — NumPy 비트 AND, 파티셔닝, 이진 탐색
- [ ] Section 7: Benchmarks — 빈 칸으로 남김 (측정 항목·시나리오만 확정)
- [ ] Section 8: Discussion — 코드북 커버리지 한계, GDELT 품질, 느슨 SIMD
- [ ] Section 9: Conclusion + GitHub link
- [ ] References — GDELT, Wikidata, WordNet, HNSW, FAISS

---

# Phase C: 데모

## 4. GDELT 파이프라인 `[C]`

GDELT 이벤트 → SIDX 이벤트 인코딩.

- [ ] BigQuery 쿼리 작성 (NumMentions >= 5, 주요 필드 추출)
- [ ] GDELT CSV → 이벤트 SIDX 변환 스크립트 (`scripts/gdelt_pipeline.py`)
- [ ] Actor → Q-ID 링킹 (Tier 1: 국가 261개, Tier 2: 주요 기관 500개)
- [ ] 이벤트 SIDX 인코딩 (entity_type=61 Event용 attrs 48비트 스키마 설계)
- [ ] 이벤트 인덱스 저장 (`index/event_sidx.npy` + `index/event_meta.npy`)

## 6. 쿼리 + 노트북 `[C]`

사용자 대면 데모. Jupyter 노트북 + 임의 프롬프트.

- [ ] 베스트 질문 10개 리스트 (시나리오별 1개씩)
- [ ] 코드북 룩업 기반 쿼리 파서 (Level 1: dict 매핑)
- [ ] LLM 보조 쿼리 파서 (Level 3: 모호성/복합 쿼리)
- [ ] 결과 표시: QID → 라벨 역변환 + 위키데이터 링크
- [ ] `notebooks/01_entity_search.ipynb` — 엔티티 검색 데모
- [ ] `notebooks/02_event_search.ipynb` — 이벤트 검색 데모
- [ ] `notebooks/03_relation_search.ipynb` — 관계 검색 데모 (교집합)
- [ ] `notebooks/04_free_query.ipynb` — 사용자 임의 프롬프트 데모

---

# Phase D: 벤치마크

## 7. 벤치마크 `[D]`

FAISS 대비 비교. 논문 Section 7 채우기.

- [ ] 임베딩 모델 선정 (예: `all-MiniLM-L6-v2` 384dim, sentence-transformers)
- [ ] FAISS 베이스라인 구축 (faiss-cpu, IVF or HNSW)
- [ ] Ground truth 정의 — SILK와 FAISS 비교를 위한 정답셋 구축 방법
  - 시나리오별 수동 정답 큐레이션 또는 위키데이터 SPARQL 기반
- [ ] 10개 시나리오 정의 (`benchmarks/scenarios.json`)
  - 엔티티 3개: 한국 정치인, 오리온자리 항성, 멸종위기 포유류
  - 이벤트 4개: 미국 대중국 외교, 머스크 관련, 한일 갈등, 러-우 군사
  - 관계 3개: 오바마-시진핑 회동, 삼성 국제 이벤트, G7 협력
- [ ] 측정 항목: 검색 시간, 인덱스 크기, 메모리 사용량, Recall@k
- [ ] `benchmarks/run_silk.py` — SILK 벤치마크 실행
- [ ] `benchmarks/run_faiss.py` — FAISS 벤치마크 실행
- [ ] Pinecone 비교 (선택 — CLAUDE.md 목표에 언급)
- [ ] 결과 시각화 (`notebooks/05_benchmarks.ipynb`)

---

# Phase E: 마무리

## 8. 논문 완성 + 배포 `[E]`

벤치마크 숫자로 논문 빈 칸 채우기. 동시에 배포 준비.

### 논문 마무리
- [ ] Abstract — 핵심 숫자 (108.8M 엔티티, N억 이벤트, ~1.3GB, 검색 시간)
- [ ] Section 7 벤치마크 결과 삽입 (Table, Figure)
- [ ] 전체 교정 + arXiv 제출

### 배포
- [ ] README.md — 설치, 퀵스타트, 시나리오 예시
- [ ] `scripts/download_index.py` — 사전 빌드된 인덱스 다운로드 (GitHub Release 또는 HuggingFace)
- [ ] 인덱스 없이 소규모 데모 가능한 모드 (샘플 10K개 내장)
- [ ] GitHub Actions CI (lint + 소규모 테스트)

---

## 의존 관계

```
GEUL 하위 프로젝트 (외부) → data/ 산출물
    ↓
코드북 데이터 (1) + 코어 라이브러리 (2)
    ↓
┌───────────┬──────────────┐
위키 (3)    GDELT (4)      논문 초안 (5)
└───────────┴──────────────┘
            ↓
       쿼리+노트북 (6)  ← 논문이 요구하는 시나리오 기반
            ↓
       벤치마크 (7)
            ↓
       논문 완성 + 배포 (8)
```

## 즉시 착수 가능 (블로커 없음)

1. **GEUL 하위 프로젝트 → data/**: 코드북 산출물 배치 (GEUL 하위 프로젝트 프로젝트 선행)
2. **`scripts/build_index.py`**: entity_sidx → index/ NumPy 배열 (sidx + qid)
3. **`silk/search.py`**: NumPy 로드 + 비트 AND 검색 — 가장 빠른 PoC
