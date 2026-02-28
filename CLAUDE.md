# SILK

SILK (Symbolic Index for LLM Knowledge) — 뉴로-심볼릭 검색 아키텍처.

64비트 정수로 검색한다. 벡터 DB, ANN 그래프, 임베딩 모델이 필요 없다.

## 프로젝트 목표

### 목표 1: 논문

```
"SILK: Symbolic Index for LLM Knowledge —
 Sub-Second Semantic Search over Billion-Scale
 Event Databases without Vector Embeddings"
```

- 위키데이터 108.8M 엔티티 + GDELT 수억 이벤트
- 벡터 DB (FAISS C++, Pinecone Rust) 대비 벤치마크
- 핵심 주장: Python(NumPy)으로 최적화된 C++/Rust를 이긴다 — 아키텍처의 승리

### 목표 2: 데모 (PoC)

```
GitHub 레포 공개. pip install → 5분 실행 → 재현 완료.
논문 리뷰어가 직접 돌려볼 수 있어야 한다.
```

- 10개 벤치마크 시나리오 (엔티티 3 + 이벤트 4 + 관계 3)
- Jupyter 노트북 데모
- 매핑 테이블 전부 공개 (CAMEO→워드넷, FIPS→Q-ID)
- 사용자 임의 프롬프트 지원 (베스트 질문 리스트 + 자유 입력)

## 브랜드 체계

| 용어 | 의미 | 사용 맥락 |
|------|------|----------|
| **SILK** | 검색 아키텍처 | 이 레포. NumPy 비트 AND 검색 엔진 |
| **SIDX** | 64비트 구조화 식별자 | 비트 구조, 인코딩, 개별 식별자를 지칭할 때 |
| **geul-sidx** | 코드북+인코딩 프로젝트 | 타입 정의, 코드북 빌드, 인코딩 파이프라인, VALID 검증 |
| **GEUL** | 프로젝트/브랜드 | 언어 설계, 장기 비전을 지칭할 때 |
| **VALID** | 기계 검증 단계 | 코드북 유효값/정합성 검사를 지칭할 때 |

## 기술 스택

- **언어: Python only.** Go/Rust 포팅 불필요.
- NumPy 벡터 연산이 사실상 SIMD. 병목은 메모리 대역폭이지 언어가 아니다.
- "최적화 안 해도 이긴다"가 논문의 킬러 어규먼트.

## 위키데이터 PostgreSQL

GEUL 프로젝트(`../geul/`)에서 위키데이터 전체 덤프를 PostgreSQL에 적재 완료.

### 접속 정보

| 계정 | 비밀번호 | DB | 용도 |
|------|---------|------|------|
| `geul_reader` | `test1224` | `geuldev` | 위키데이터 원본 (읽기 전용) |
| `geul_writer` | `test1224` | `geulwork` | SIDX 인코딩 작업용 (읽기/쓰기) |

```python
# 예시
import psycopg2
conn = psycopg2.connect('postgresql://geul_reader:test1224@localhost:5432/geuldev')
```

### geuldev (위키데이터 원본)

| 테이블 | 행 수 | 크기 | 용도 |
|--------|-------|------|------|
| `entities` | 117.4M | 5.8GB | Q-ID/P-ID 목록 (item 117.4M + property 12,870) |
| `triples` | 1,703M | 154GB | (subject, property, object_value) 전체 트리플 |
| `entity_labels` | — | 49GB | 다국어 라벨 |
| `entity_descriptions` | — | 232GB | 다국어 설명 |
| `entity_aliases` | — | 10GB | 다국어 별칭 |
| `properties_meta` | 12,872 | 2.3MB | 속성 메타데이터 |
| `property_usage_stats` | 12,315 | 672KB | 속성별 사용 빈도 |
| `property_object_stats` | 59.6M | 3.5GB | (속성, 목적어) 쌍 빈도 — P31 타입 통계 포함 |
| `hierarchy` | 0 | — | P31/P279 계층 (미적재) |

### geulwork (SIDX 인코딩 결과)

| 테이블 | 행 수 | 크기 | 용도 |
|--------|-------|------|------|
| `entity_sidx` | 108.8M | 9.5GB | 인코딩된 (qid, sidx, entity_type, attrs) |
| `codebook` | 142,506 | 29MB | 타입별 필드→코드 매핑 |
| `bit_allocation` | 38 | 16KB | 5개 타입의 비트 필드 배치 |
| `entity_type_map` | 63 | 16KB | 타입코드→이름 매핑 (0=Human ~ 62=Election) |
| `dependency_dag` | 671 | 88KB | 필드 간 종속 관계 |

### entity_type_map (63개 타입)

```
카테고리      타입 (코드: 이름)
─────────────────────────────────────────
생물/인물     0:Human 1:Taxon 2:Gene 3:Protein 4:CellLine 5:FamilyName 6:GivenName 7:FictionalCharacter
화학/물질     8:Chemical 9:Compound 10:Mineral 11:Drug
천체          12:Star 13:Galaxy 14:Asteroid 15:Quasar 16:Planet 17:Nebula 18:StarCluster 19:Moon
지형/자연     20:Mountain 21:Hill 22:River 23:Lake 24:Stream 25:Island 26:Bay 27:Cave
장소/행정     28:Settlement 29:Village 30:Hamlet 31:Street 32:Cemetery 33:AdminRegion 34:Park 35:ProtectedArea
건축물        36:Building 37:Church 38:School 39:House 40:Structure 41:SportsVenue 42:Castle 43:Bridge
조직          44:Organization 45:Business 46:PoliticalParty 47:SportsTeam
창작물        48:Painting 49:Document 50:LiteraryWork 51:Film 52:Album 53:MusicalWork 54:TVEpisode 55:VideoGame 56:TVSeries 57:Patent 58:Software 59:Website
이벤트        60:SportsSeason 61:Event 62:Election
미분류        63 (19.7M개)
```

### bit_allocation (인코딩 완료 5개 타입)

속성 비트 배치가 정의된 타입: **Human(0), Star(12), Settlement(28), Organization(44), Film(51)**.
나머지 58개 타입은 entity_type + QID만 인코딩됨 (attrs=0).

## SIDX 64비트 레이아웃

geulwork.entity_sidx에 108.8M개 인코딩 완료된 실제 레이아웃.

```
word1 = (PREFIX << 9) | (mode << 6) | entity_type
sidx  = (word1 << 48) | attrs

[prefix 7 | mode 3 | entity_type 6 | attrs 48]
 MSB(63)                                  LSB(0)

prefix:      7비트 — GEUL 프로토콜 헤더 (고정값 0b0001001, 검색 시 무시)
mode:        3비트 — 등록 모드 (PoC에서는 항상 0)
entity_type: 6비트 — 63개 타입 (Human=0 ~ Election=62, 미분류=63)
attrs:      48비트 — 타입별 속성 인코딩 (bit_allocation 정의)

Q-ID는 sidx에 포함되지 않음. 별도 컬럼(qid)에 저장.
동일한 분류의 엔티티는 동일한 sidx 값을 가질 수 있다.
```

### 검색에 쓰는 비트

prefix/mode는 전 엔티티 동일 → 검색에 무의미.
실제 검색 대상: **entity_type 6비트 + attrs 48비트 = 54비트.**

```python
TYPE_MASK    = 0x003F_0000_0000_0000   # bits 53-48
type_pattern = entity_type << 48

# attrs 내 필드는 bit_allocation의 (offset, width) 그대로
attr_mask    = ((1 << width) - 1) << offset
attr_pattern = code << offset

mask    = TYPE_MASK | attr_mask
pattern = type_pattern | attr_pattern
hits    = index[(index & mask) == pattern]
```

### attrs 48비트 — 타입별 스키마 (bit_allocation)

```
Human(0):        subclass 5 | occupation 6 | country 8 | era 4 | decade 4 | gender 2 | notability 3 | reserved 16
Star(12):        constellation 7 | spectral_type 4 | luminosity 3 | magnitude 4 | ra_zone 4 | dec_zone 4 | flags 6 | reserved 16
Settlement(28):  country 8 | admin_level 4 | admin_code 8 | lat_zone 4 | lon_zone 4 | population 4 | timezone 5 | reserved 11
Organization(44): country 8 | org_type 4 | legal_form 6 | industry 8 | era 4 | size 4 | reserved 14
Film(51):        country 8 | year 7 | genre 6 | language 8 | color 2 | duration 4 | reserved 13
```

나머지 58개 타입은 attrs=0 (entity_type만 인코딩됨).

### 인덱스 구조

```python
# sidx 배열 + QID 배열 (병렬)
sidx_array = np.array([...], dtype=np.uint64)   # 108.8M × 8B = 870MB
qid_array  = np.array([...], dtype=np.uint32)   # 108.8M × 4B = 435MB
# 총 ~1.3GB 메모리

hits = (sidx_array & mask) == pattern
result_qids = qid_array[hits]
```

## Multi-SIDX

하나의 문서/이벤트에 여러 개의 SIDX가 붙는다.
entity_type 필드가 엔티티(Human, Organization)와 이벤트(Event)를 구분.
전부 같은 64비트 SIDX. 같은 인덱스. 같은 NumPy 비트 AND로 검색.

## 검색

```python
# 쿼리 → mask/pattern 조립 → Python 한 줄
results = index[(index & mask) == pattern]
```

```
검색 파이프라인:
  1. 쿼리 의미 추출     (코드북 룩업 80% / LLM 보조 20%)
  2. 비트마스크 조립     (결정적 — 알고리즘)
  3. NumPy 비트 AND     (결정적 — 벡터 연산)
  4. LLM 최종 판정      (소수 후보만)
```

## 아키텍처

```
심볼릭 (규칙)              뉴럴 (LLM)
├── 코드북 yaml             ├── 소형 LLM 태깅
├── VALID 검증              ├── 대형 LLM 검수
├── NumPy 비트 AND          └── LLM 최종 판정
└── 정렬 + 이진 탐색

각자 잘하는 것만 수행:
  사람: 구조 설계 (코드북)
  LLM:  자연어 분류 (태깅) + 의미 추출 (쿼리)
  코드: 규칙 검증 (VALID) + 비트 조립
  CPU:  대량 비교 (NumPy)
```

## 코드북 (geul-sidx)

코드북 산출은 별도 프로젝트 **geul-sidx** 범위. SILK은 산출물만 소비.

```
geul-sidx (별도 프로젝트)              silk (이 레포)
├── 타입 정의, P279 체인               ├── data/  ← geul-sidx 산출물
├── 코드북 빌드, 커버리지 개선         ├── silk/  ← 검색 엔진
├── 인코딩 파이프라인                  ├── index/ ← 인코딩된 인덱스
├── 동사 코드북 (WordNet)              └── benchmarks/
└── VALID 검증
```

현황:
- 동사 코드북: 완성 (WordNet 13,767 synset → 16비트)
- 엔티티 코드북: 142,506 엔트리, 5개 타입 속성 인코딩 완료
- 미분류 19.7M개 (entity_type=63) — geul-sidx에서 해결

## 디렉토리 구조

```
silk/
├── CLAUDE.md
├── pyproject.toml
├── .gitignore             # draft/, backup/, index/ 제외
│
├── silk/                  # 코어 라이브러리 (5개 파일)
│   ├── codebook.py        # 코드북 로드, lookup, 비트 조립
│   ├── encoder.py         # tag → SIDX uint64
│   ├── valid.py           # VALID 검증
│   ├── search.py          # NumPy 비트 AND
│   └── query.py           # 쿼리 → mask/pattern
│
├── data/                  # geul-sidx 산출물 (코드북, 매핑 테이블)
├── scripts/               # 데이터 파이프라인 (1회성)
├── index/                 # 생성된 인덱스 (gitignored)
├── benchmarks/            # 벤치마크 (FAISS 비교)
├── notebooks/             # Jupyter 데모
├── paper/                 # 논문
├── draft/                 # 설계 초안 (gitignored)
└── backup/                # 비활성 문서 (gitignored)
```

## draft 문서

| 파일 | 내용 |
|------|------|
| `SILK-개요.md` | 아키텍처 전체 개요. 핵심 명제, 비트 레이아웃, 탐색 5단계, 학술 포지셔닝. |
| `SILK-인코딩.md` | SIDX 인코딩 방법. 코드북 yaml, LLM 태깅 파이프라인, 비트 조립, VALID 검증. |
| `SILK-검수.md` | 3단계 검수 파이프라인. 소형 LLM 태깅 → VALID 기계 검사 → 대형 LLM 검수. |
| `SILK-탐색.md` | 탐색 아키텍처. NumPy 전수 스캔, 디렉토리 파티셔닝, 이진 탐색, LLM 판정. |
| `SILK-데모.md` | 데모 계획. Wiki + GDELT, CAMEO→워드넷 매핑, 10개 시나리오, 논문 구조. |
| `SILK-코드북.md` | 코드북 현황. 동사 코드북 완성, 엔티티 코드북 진행 중. |
| `SILK-쿼리.md` | 쿼리 생성 전략. 코드북 룩업 + LLM 보조 하이브리드, 사용자 임의 프롬프트. |

## 작업 원칙

- SILK은 검색만 한다. 코드북/인코딩은 geul-sidx 프로젝트 범위.
- CLAUDE.md가 SIDX 레이아웃의 기준. draft/ 문서는 서술적 참고용.
- SIDX = 16비트 헤더(prefix+mode+type) + 48비트 속성. QID는 별도.
- 코드는 Python. NumPy 벡터 연산 활용. C 확장 금지.
- 인덱스 = sidx uint64 배열 + qid uint32 배열 (병렬). 자료구조 없음.
- 검색 = `index[(index & mask) == pattern]`. 이게 전부.
- GEUL prefix/mode는 검색에 무관. PoC에서는 entity_type + attrs만 사용.
- 커밋 메시지에 Co-Authored-By 트레일러를 넣지 않는다.
