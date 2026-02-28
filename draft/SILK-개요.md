# SILK: Symbolic Index for LLM Knowledge

*Neuro-Symbolic Search Architecture*

---

## 한 줄 정의

```
SILK는 64비트 정수로 검색하는 뉴로-심볼릭 아키텍처다.
벡터 DB, ANN 그래프, 임베딩 모델이 필요 없다.
```

---

## 핵심 명제

```
검색은 어려운 문제가 아니라 구조 없는 데이터의 증상이었고,
기록 시점에 64비트 구조를 부여하면 증상이 사라진다.
```

---

## 이름의 의미

```
SILK = Symbolic Index for LLM Knowledge

Symbolic:  심볼릭 AI. 코드북, 비트 레이아웃, 규칙 기반 검증.
Index:     SIDX. 64비트 구조화 식별자.
LLM:       태깅, 분류, 검수를 수행하는 뉴럴 컴포넌트.
Knowledge: 지식. 검색 대상이자 구조화 대상.
```

---

## 브랜드 체계

```
GEUL:  프로젝트/브랜드명. geul.org. Grounded Encoding for Universal Language.
SIDX:  기술 스펙. 64비트 구조화 식별자. Semantic IDentifier indeX.
SILK:  아키텍처 이름. 사람들 입에 붙는 것.

"SILK 써봤어?" ← 이걸 위한 이름.
```

---

## 문제 인식

### 벡터 임베딩은 구조를 뭉개는 행위다

```
사람이 쓴 글:
  "이순신은 조선 중기의 무신이다"
  → 사람은 즉시 파악: 한국인, 군인, 16세기

임베딩 모델이 하는 일:
  [0.234, -0.891, 0.445, ..., 0.112]  (384차원 float)
  → 구조 소멸. person인지 location인지 읽을 수 없음.

그리고 뭉개진 구조를 복원하려고:
  ANN 그래프 (HNSW, IVF-PQ)
  크로스 인코더
  리랭커
  메타데이터 필터
  → 수십 년간 수천 편의 논문. 수백 개의 스타트업.
```

```
SILK:
  [person / military / east_asia / early_modern | Q484523]
  → 구조가 비트에 보존됨. 읽을 수 있음.
  → 복원할 필요 없음. 안 뭉갰으니까.
```

### 검색 산업 전체가 잘못된 순서의 산물이다

```
기존:  먼저 쓰고 → 나중에 구조화 (인덱싱)
SILK:  쓸 때 구조화 → 검색이 공짜

Elasticsearch, Pinecone, FAISS, Weaviate, Qdrant,
BM25, TF-IDF, HNSW, IVF-PQ, 벡터 RAG...
전부 "먼저 쓰고 나중에 찾기"의 결과물.
```

---

## 아키텍처

### 뉴로-심볼릭 구조

```
┌─────────────────────────────────────────────┐
│                  SILK                         │
│                                               │
│   심볼릭                    뉴럴              │
│   ┌─────────────┐          ┌──────────────┐  │
│   │ 코드북 yaml  │          │ LLM 태깅     │  │
│   │ 비트 레이아웃│          │ LLM 분류     │  │
│   │ VALID 검증  │    JSON   │ LLM 검수     │  │
│   │ SIMD 스캔   │◄────────►│              │  │
│   │ Q-ID 식별   │          │              │  │
│   └─────────────┘          └──────────────┘  │
│                                               │
│   각자 잘하는 것만 수행:                       │
│     사람: 구조 설계 (코드북)                   │
│     LLM:  자연어 분류 (태깅)                   │
│     코드: 규칙 검증 (VALID)                    │
│     CPU:  대량 비교 (SIMD)                     │
└─────────────────────────────────────────────┘
```

### SIDX 64비트 레이아웃

```
sidx = (type << 58) | (subtype << 53) | (region << 49) | (era << 46) | qid

[type 6 | subtype 5 | region 4 | era 3 | 미사용 14 | Q-ID 32]
 MSB                                                        LSB

type:    6비트 (64가지)  — 위키데이터 P31 상위 64개
subtype: 5비트 (32가지)  — 타입별 32분류
region:  4비트 (16권역)  — 대륙/권역
era:     3비트 (8시대)   — 시대구간
미사용:  14비트           — 0
Q-ID:   32비트           — 위키데이터 Q-ID / 도메인 PK
```

18비트 분류 + 32비트 식별자. 하나의 uint64 안에 필터링 + 식별 전부.
대강 짜도 108M / 262,144 = 412건/버킷. 충분.

### Multi-SIDX

하나의 문서에 여러 개의 SIDX가 붙는다.

```
뉴스 기사 "삼성·엔비디아·현대차 총수 회동":

SIDX[0]: [person  / business   / east_asia  / current | Q484523  ]  이재용
SIDX[1]: [org     / company    / east_asia  / current | Q81965   ]  삼성전자
SIDX[2]: [person  / business   / n_america  / current | Q7352287 ]  젠슨황
SIDX[3]: [org     / company    / n_america  / current | Q182477  ]  엔비디아
SIDX[4]: [person  / business   / east_asia  / current | Q20895436]  정의선
SIDX[5]: [org     / company    / east_asia  / current | Q55043   ]  현대차
SIDX[6]: [event   / meeting    / east_asia  / current | 0        ]  회동
```

전부 같은 64비트 SIDX. 같은 인덱스. 같은 SIMD로 검색.
type 필드가 엔티티(person, org)와 이벤트(event)를 구분.

### 인덱스 = 정렬된 (SIDX, ptr) pair 배열

```
인덱스 전체:
  [(SIDX, ptr), (SIDX, ptr), (SIDX, ptr), ...]

구축 방법: sort.
자료구조: 없음. 배열 하나.
```

```
Elasticsearch: 토크나이징 → 분석 → 역인덱스 → 세그먼트 머지
Pinecone:      임베딩 → HNSW 그래프 → 클러스터링
SILK:          sort
```

---

## 탐색

### 5단계 최적화

SIDX의 비트 레이아웃이 물리적 데이터 배치와 일치하므로
단계별로 탐색 범위가 기하급수적으로 줄어든다.

#### 0단계: 전수 스캔 (NumPy)

```python
# "한국 권역 정치인" 검색
mask    = 0b_111111_11111_1111_000_00000000000000_00000000000000000000000000000000
pattern = 0b_000000_00000_0000_000_00000000000000_00000000000000000000000000000000
#           type=0  sub=0 reg=0  (실제값 대입)

# Python 한 줄
results = index[(index & mask) == pattern]
```

NumPy 벡터 연산이 사실상 SIMD.
병목은 메모리 대역폭. 연산 자체는 공짜.

#### 1단계: 디렉토리 파티셔닝

```
비트 상위→하위 = 의미 대분류→소분류 = 디렉토리 깊이.
파일을 여는 행위가 필터링. 비트 AND 0번.

index/
├── person/
│   ├── politician/
│   │   ├── east_asia.geul
│   │   └── north_america.geul
│   ├── military/
│   └── business/
├── organization/
│   ├── company/
│   └── university/
├── event/
│   ├── meeting/
│   └── diplomacy/
├── star/
└── ...
```

```
         기준            파일 수      파일 크기 (1조)   라즈베리파이
깊이 0:  없음            1           8TB              5시간
깊이 1:  type 6비트      64          125GB            7분
깊이 2:  + subtype 5비트 2,048       3.9GB            8초
깊이 3:  + region 4비트  32,768      244MB            0.5초
```

#### 2단계: 정렬 + 이진 탐색

```
파일 내부가 Q-ID 순 정렬:
  전수 스캔 → 이진 탐색. log2(900만) = 23번 비교.

900만건 전수: 72MB 읽기, 144ms.
이진 탐색:     184바이트 읽기, 0.002ms.
```

#### 3단계: doc_id도 정렬

```
각 Q-ID의 doc_id 리스트가 정렬되어 있으므로
교집합 = merge join. 포인터 두 개 순회. 추가 메모리 0.
```

#### 4단계: (SIDX, ptr) pair 정렬

```
SIDX 순 정렬 → "이 조건에 맞는 문서 전부" (검색용)
doc_id 순 정렬 → "이 문서의 태그 전부" (조회용)
같은 데이터, 정렬만 다름.
```

### 1조 SIDX 탐색 성능 종합

```
| 단계 | 읽는 데이터 | 라즈베리파이 | 서버 |
|------|-----------|------------|------|
| 전수 스캔 | 8TB | 5시간 | 140ms |
| type 파티셔닝 | 125GB | 7분 | 2.5초|
| type+subtype | 3.9GB | 8초 | 80ms |
| +region | 244MB | 0.5초 | 5ms |
| +이진탐색 | 100KB | 1ms | 0.01ms |
```

5시간 → 1ms. 같은 데이터. 같은 하드웨어. 같은 결과.

### 분할 가능성

```
비트 AND는 순서 무관, 상태 없음, 합산 가능.
부분의 합이 전체와 같다 (집합 연산).

벡터 DB: 그래프 전체가 메모리에 있어야 함. 분할 = 파괴.
SILK:    어디서 잘라도 작동. 분할 = 느려질 뿐. 결과 동일.
```

```
| 장비 | 비용 | 1조 탐색 | 용도 |
|------|------|---------|------|
| 라즈베리파이 | $35 | 1ms (최적화) | 연구/학생 |
| 노트북 | $1000 | 0.1ms | 개인 |
| AWS 12대 온디맨드 | $2.90/회 | 140ms (전수) | 대기업 |
| AWS 200대 스팟 | $0.87/회 | 800ms (전수) | 빅테크 배치 |
| H100 104장 | $400K/월 | 24ms (전수) | 실시간 서비스 |

전부 같은 코드. 같은 바이너리. 같은 결과.
```

---

## 검수 파이프라인

### 벡터는 검수 불가능. SILK는 JSON이다.

```
벡터 임베딩: [0.234, -0.891, 0.445, ..., 0.112]
  → 맞는지 판단 기준 없음. 블랙박스.
  → 사람 검수 불가. LLM 검수 불가. 기계 검수 불가.

SILK SIDX:   {"type": "person", "subtype": "military", "region": "east_asia"}
  → 읽을 수 있음. 화이트박스.
  → 사람 검수 가능. LLM 검수 가능. 기계 검수 가능.
```

### 3단계 품질 보증

```
1단계 — 소형 LLM 대량 태깅
  모델: Llama 8B / GPT-4o-mini
  비용: $0.00003/건. 1.08억건 = $3,240.
  정확도: 85-90%.

2단계 — VALID 기계 검사
  유효값 체크: 코드북에 있는 값인가?
  정합성 체크: person인데 subtype이 oil_painting?
  제약조건 체크: star인데 region이 east_asia?
  비용: $0. if문.
  코드북이 문지기. 유효값 바깥의 환각은 물리적으로 인덱스에 못 들어감.

3단계 — 대형 LLM 검수
  모델: Claude Opus / GPT-4
  대상: confidence=low인 필드만 (스마트 검수)
  비용: $59,400 (전수 $324,000 대비 82% 절약)
  정확도: 99%+.
```

### 환각 방어

```
| 환각 패턴 | 방어 주체 | 단계 |
|----------|----------|------|
| 없는 키워드 ("admiral") | VALID 유효값 체크 | 2단계 |
| 타입 혼동 (person/oil_painting) | VALID 정합성 체크 | 2단계 |
| 과잉 구체화 ("16th_century_joseon...") | VALID 유효값 체크 | 2단계 |
| 애매한 분류 (군인을 artist로) | 대형 LLM 의미 검수 | 3단계 |
| Q-ID 오매칭 | 위키데이터 lookup | 2.5단계 |
| JSON 깨짐 | JSON 파서 | 0단계 |
| 필드 누락 | VALID 필수 필드 체크 | 2단계 |

7개 패턴 중 5개 VALID 자동 차단.
인덱스 오염 가능성: 0.
```

---

## RAG를 넘어서

### 검색의 80%는 구조적 쿼리다

```
"이순신" → Q484523 exact match. LLM 검수 불필요. 수학적 확정.
"삼성 뉴스" → org/company/Q81965 + doc_meta/news. 비트 AND. 끝.
"바이든-시진핑 회담" → Q6279 ∩ Q15031 ∩ meeting. 교집합. 끝.
```

```
구조적 쿼리 (80%): SILK 비트 AND로 완결. LLM 불필요.
의미적 쿼리 (15%): SILK로 후보 축소 → LLM 5-10건 경량 판정.
생성 쿼리 (5%):   SILK로 문서 특정 → LLM 생성. 여기서만 RAG적.
```

### SILK는 RAG의 대체제가 아니다

```
RAG가 묻는 질문: "어떻게 더 잘 검색할까?"
SILK가 묻는 질문: "검색이 왜 필요하지?"

SILK는 대부분의 검색에서 RAG가 필요 없었다는 증명이다.
```

### 벡터 DB 비교

```
| | SILK | 벡터 DB |
|------|------|---------|
| 인덱스 크기 (1조) | 12TB | 1.5PB (125배) |
| 인덱스 구축 | sort | HNSW 수 일 |
| 콜드 스타트 | 파일 열면 즉시 | 그래프 구축 수 시간 |
| 분할 스캔 | 가능 (결과 동일) | 불가능 (그래프 끊김) |
| 복합 조건 | 교집합 (정확) | 벡터 1개로 압축 (근사) |
| 결과 | 정확 (집합 연산) | 근사 (유사도 순위) |
| 디렉토리 파티셔닝 | 가능 | 불가능 |
| 이진 탐색 | 가능 | 불가능 |
| 검수 | JSON (화이트박스) | 불가능 (블랙박스) |
```

---

## uint64 하나가 전부

```
하나의 SIDX (uint64):

데이터로:   person/military/east_asia/early_modern/Q484523 — 즉시 해석
코드로:     (x & mask) == pattern — 실행 가능한 필터
인덱스로:   NumPy AND 한 번 — 1억건 20ms 검색
파일로:     정렬된 배열 레코드 하나 — 랜덤 액세스 O(1)
언어로:     한국어=영어=일본어 — 보편 식별자

변환 레이어: 0개.
```

```
기존: 토크나이저 + 직렬화 + 파일포맷 + DB + 검색엔진 + 쿼리언어 + NLP
     = 7개 시스템. 6개 변환.

SILK: uint64 배열. 비트 AND.
     = 끝.
```

---

## 학술 포지셔닝

### Neuro-Symbolic AI

```
심볼릭 AI (1956~):
  장점: 해석 가능, 검증 가능, 조합 가능
  단점: 스케일링 불가. 규칙을 사람이 다 짜야 함.
  → AI 겨울.

뉴럴 AI (2012~):
  장점: 스케일링 가능. 데이터에서 패턴 학습.
  단점: 블랙박스. 검증 불가. 환각.

SILK (2026):
  규칙 정의: 사람 (코드북 yaml)     ← 적은 수
  규칙 적용: LLM (태깅)             ← 스케일 가능
  규칙 검증: 코드 (VALID)           ← 자동
  = 심볼릭의 모든 장점 + 뉴럴의 스케일링.
```

심볼릭이 실패한 건 심볼릭이 나빠서가 아니라 스케일링 수단이 없어서. LLM이 스케일링해주면 심볼릭이 부활한다.

### 패러다임 전환

```
기존 검색: Ranking (순위 매기기)
  "가장 관련 있는 top-10을 찾아라"
  → 벡터 유사도 → ANN → NDCG@10 평가

SILK: Classification (분류)
  "조건에 맞는 전부를 걸러라"
  → 비트 AND → 교집합 → Recall@all 평가

ranking은 근사. classification은 정확.
```

---

## 시작하기

```sql
ALTER TABLE entities ADD COLUMN sidx BIGINT;
```

---

## 삼각 편대

```
geul.org/why-silk:   "검색은 어려운 문제가 아니었다"     (왜?)
논문:                "SILK: Symbolic Index for LLM Knowledge" (어떻게?)
github.com/geul/silk: GDELT-WIKI 데모, 5분 실행          (진짜?)
```

---

## 한 문장

```
SILK. Neuro-symbolic search that replaces vector DBs 
with 64-bit integers. 1 trillion entries, 1ms, 
on a Raspberry Pi.
```
