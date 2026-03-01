# SILK

**Symbolic Index for LLM Knowledge** — 뉴로-심볼릭 검색 아키텍처.

64비트 정수로 검색한다. 벡터 DB, ANN 그래프, 임베딩 모델이 필요 없다.

[GEUL](https://geul.org) 프로젝트의 하위 프로젝트.

## 핵심 명제

검색은 어려운 문제가 아니라 **구조 없는 데이터의 증상**이었고,
기록 시점에 64비트 구조를 부여하면 증상이 사라진다.

```python
results = index[(index & mask) == pattern]
```

위키데이터 1억 개 엔티티를 1.3GB 메모리에서 서브초 검색.
Python(NumPy)만으로 최적화된 C++/Rust 벡터 DB를 이긴다 — 아키텍처의 승리.

## 왜 벡터 임베딩이 아닌가

벡터 임베딩은 구조를 뭉갠다.

```
"이순신은 조선 중기의 무신이다"
  → 사람은 즉시 파악: 한국인, 군인, 16세기

임베딩 모델:
  [0.234, -0.891, 0.445, ..., 0.112]  (384차원 float)
  → 구조 소멸. person인지 location인지 읽을 수 없음.

뭉개진 구조를 복원하려고:
  ANN 그래프 (HNSW, IVF-PQ), 크로스 인코더, 리랭커, 메타데이터 필터...
```

SILK은 구조를 보존한다.

```
SIDX: [Human / military / east_asia / early_modern]
  → 비트에 구조가 살아 있음. 읽을 수 있음.
  → 복원할 필요 없음. 안 뭉갰으니까.
```

핵심은 순서의 전환이다.

```
기존:  먼저 쓰고 → 나중에 구조화 (인덱싱)
SILK:  쓸 때 구조화 → 검색이 공짜
```

## SIDX 비트 레이아웃

SIDX는 [GEUL 표준](https://geul.org/grammar/) Entity Node 규격을 따른다.

```
[prefix 7 | mode 3 | entity_type 6 | attrs 48]
 MSB(63)                                  LSB(0)
```

| 필드 | 비트 | 크기 | 설명 |
|------|------|------|------|
| prefix | 63-57 | 7 | GEUL 프로토콜 헤더 (`0001001` 고정, 검색 시 무시) |
| mode | 56-54 | 3 | 양화/수 모드 (등록 엔티티=0, 정관사, 보편, 존재 등 8가지) |
| entity_type | 53-48 | 6 | 64개 최상위 타입 (Human=0 ~ Election=62, 미분류=63) |
| attrs | 47-0 | 48 | 타입별 속성 인코딩 (코드북 정의) |

검색 대상: entity_type 6비트 + attrs 48비트 = **54비트**.
QID는 SIDX에 포함되지 않으며 별도 배열에 저장한다.

### attrs 48비트 — 타입별 스키마

```
Human(0):        subclass 5 | occupation 6 | country 8 | era 4 | decade 4 | gender 2 | notability 3 | ...
Star(12):        constellation 7 | spectral_type 4 | luminosity 3 | magnitude 4 | ra_zone 4 | dec_zone 4 | ...
Settlement(28):  country 8 | admin_level 4 | admin_code 8 | lat_zone 4 | lon_zone 4 | population 4 | ...
Organization(44): country 8 | org_type 4 | legal_form 6 | industry 8 | era 4 | size 4 | ...
Film(51):        country 8 | year 7 | genre 6 | language 8 | color 2 | duration 4 | ...
```

현재 5개 타입의 속성 비트 배치가 정의됨. 나머지는 entity_type만 인코딩.

## 아키텍처

SILK에는 두 개의 파이프라인이 있다. **인코딩**(쓸 때)과 **검색**(찾을 때).
둘 다 같은 원리: 심볼릭이 구조를 잡고, LLM이 의미를 처리한다.

```
인코딩 파이프라인 (쓸 때)
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  LLM 태깅     │───►│ VALID 검증    │───►│ 코드북 인코딩 │
│  문서 → JSON  │    │ 코드북 유효값  │    │ JSON → SIDX  │
│  (의미 분류)   │    │ 정합성 체크   │    │ (비트 조립)   │
│              │    │ 환각 → 탈락   │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
  LLM이 태깅         코드가 검증         코드북 기반 인코딩

검색 파이프라인 (찾을 때)
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  쿼리 해석    │───►│ 비트 AND 필터 │───►│  LLM 판정    │
│  코드북 룩업   │    │ NumPy SIMD   │    │  소수 후보만  │
│  + LLM 보조   │    │ 1억건 → 수십건│    │  수십건 → 정답│
└──────────────┘    └──────────────┘    └──────────────┘
  의미 추출           넓게 필터링          좁혀서 판정
```

핵심 전략: **SIDX 비트 AND로 놓치는 것 없이 넓게 필터링**한 다음, **LLM이 소수 후보만 판정**한다.

```
각자 잘하는 것만 수행:
  사람: 구조 설계 (코드북)
  LLM:  의미 분류 (태깅) + 의미 판정 (검색)
  코드: 규칙 검증 (VALID) + 비트 조립
  CPU:  대량 비교 (NumPy SIMD)
```

### 인코딩: LLM 태깅 → VALID 검증

LLM이 문서를 읽고 JSON으로 태깅한다. VALID이 코드북 기준으로 기계적 검사를 수행한다.
코드북에 없는 값, 타입 간 정합성 위반, 제약조건 위반은 **물리적으로 인덱스에 못 들어간다.**

```
LLM 태깅:  {"type": "Human", "occupation": "military", "country": "korea"}
VALID:     occupation="military" ∈ 코드북? ✓  country="korea" ∈ 코드북? ✓
           Human인데 constellation 필드? ✗ 탈락
인코딩:    코드북에서 "military"→6비트, "korea"→8비트 → 비트 조립 → SIDX uint64
```

LLM은 환각할 수 있다. 하지만 VALID이 문지기 역할을 하므로 인덱스 오염 가능성은 0이다.
VALID을 통과한 JSON만 코드북 기반으로 SIDX 64비트 정수로 인코딩된다.

### 검색: 넓게 필터링, LLM이 좁힌다

```
1. 쿼리 의미 추출     코드북 룩업 80% / LLM 보조 20%
2. 비트마스크 조립     결정적 — 알고리즘
3. NumPy 비트 AND     결정적 — 1억건 전수 스캔 20ms
4. LLM 최종 판정      소수 후보만 — 의미적 정밀 판정
```

### 검색의 80%는 구조적 쿼리다

```
"이순신"           → Q484523 exact match. 비트 AND. 끝.
"삼성 뉴스"        → org/company + doc_meta/news. 비트 AND. 끝.
"바이든-시진핑 회담" → Q6279 ∩ Q15031 ∩ meeting. 교집합. 끝.
```

```
구조적 쿼리 (80%):  SILK 비트 AND로 완결. LLM 불필요.
의미적 쿼리 (15%):  SILK로 후보 축소 → LLM 5-10건 판정.
생성 쿼리 (5%):     SILK로 문서 특정 → LLM 생성.
```

## Multi-SIDX

하나의 문서/이벤트에 여러 개의 SIDX가 붙는다.

```
뉴스 기사 "삼성·엔비디아·현대차 총수 회동":

SIDX[0]: [Human / business  / east_asia ]  이재용
SIDX[1]: [Org   / company   / east_asia ]  삼성전자
SIDX[2]: [Human / business  / n_america ]  젠슨황
SIDX[3]: [Org   / company   / n_america ]  엔비디아
SIDX[4]: [Human / business  / east_asia ]  정의선
SIDX[5]: [Org   / company   / east_asia ]  현대차
SIDX[6]: [Event / meeting   / east_asia ]  회동
```

전부 같은 64비트 SIDX. 같은 인덱스. 같은 비트 AND로 검색.
entity_type 필드가 엔티티(Human, Org)와 이벤트(Event)를 구분.

## 인덱스 구조

```python
sidx_array = np.array([...], dtype=np.uint64)   # 108.8M × 8B = 870MB
qid_array  = np.array([...], dtype=np.uint32)   # 108.8M × 4B = 435MB
# 총 ~1.3GB 메모리
```

```
인덱스 구축:
  Elasticsearch: 토크나이징 → 분석 → 역인덱스 → 세그먼트 머지
  Pinecone:      임베딩 → HNSW 그래프 → 클러스터링
  SILK:          sort
```

자료구조 없음. 정렬된 배열 하나.

## 벡터 DB 비교

| | SILK | 벡터 DB |
|------|------|---------|
| 인덱스 크기 (1조건) | 12TB | 1.5PB (125배) |
| 인덱스 구축 | sort | HNSW 수일 |
| 콜드 스타트 | 파일 열면 즉시 | 그래프 구축 수 시간 |
| 분할 스캔 | 가능 (결과 동일) | 불가능 (그래프 끊김) |
| 복합 조건 | 교집합 (정확) | 벡터 1개로 압축 (근사) |
| 결과 | 정확 (집합 연산) | 근사 (유사도 순위) |
| 검수 | JSON (화이트박스) | 불가능 (블랙박스) |

```
비트 AND는 순서 무관, 상태 없음, 합산 가능.
벡터 DB: 그래프 전체가 메모리에 있어야 함. 분할 = 파괴.
SILK:    어디서 잘라도 작동. 분할 = 느려질 뿐. 결과 동일.
```

## 검수 파이프라인

벡터 임베딩은 블랙박스라 검수 불가능. SIDX는 JSON이라 검수 가능.

```
1단계 — 소형 LLM 태깅     Llama 8B / GPT-4o-mini. 정확도 85-90%.
2단계 — VALID 기계 검사    유효값, 정합성, 제약조건. 비용 $0.
3단계 — 대형 LLM 검수      confidence=low만. 정확도 99%+.

VALID이 문지기: 코드북 유효값 바깥의 환각은 물리적으로 인덱스에 못 들어감.
```

## 설치

```bash
pip install -e .
```

### 의존성

- Python ≥ 3.10
- NumPy

## 프로젝트 구조

```
silk/
├── silk/           # 코어 라이브러리
│   ├── codebook.py # 코드북 로드, lookup, 비트 조립
│   ├── encoder.py  # tag → SIDX uint64
│   ├── valid.py    # VALID 검증
│   ├── search.py   # NumPy 비트 AND
│   └── query.py    # 쿼리 → mask/pattern
├── data/           # 코드북, 매핑 테이블 (GEUL 산출물)
├── scripts/        # 데이터 파이프라인
├── index/          # 생성된 인덱스 (gitignored)
├── benchmarks/     # FAISS 비교 벤치마크
├── notebooks/      # Jupyter 데모
└── paper/          # 논문
```

## 라이선스

MIT
