# SILK 데모: Wiki + GDELT

*2026년 2월 27일*

---

## 한 줄 요약

위키데이터 108.8M 엔티티 + GDELT 수십억 이벤트를 SILK SIDX로 인코딩하고, 아무 질문이나 150ms에 답한다. 임베딩 없이. 벡터 DB 없이. 노트북에서.

---

## 왜 GDELT인가

CC NEWS 10억건을 직접 정규화하려면 NER + Event Extraction + Entity Linking을 처음부터 해야 한다. GPU 서버 1-2주, 비용 $1,000-5,000.

GDELT는 이 작업을 1979년부터 46년간 해왔다. 15분마다 전 세계 뉴스를 수집하고, 자동으로 이벤트를 추출하고, 구조화해서 공개한다.

```
우리가 안 해도 되는 것:  NER, Event Extraction, 시간/장소 정규화
우리가 해야 할 것:       코드 체계 변환 (CAMEO → 워드넷, Actor → Q-ID)
```

작업량이 1/10로 줄었다.

### 라이센스

100% 무료. 학술, 상업, 정부 용도 무제한 사용 가능. 재배포, 재호스팅, 재발행 허용. 조건은 출처 표기("GDELT Project" + URL)뿐.

AWS Open Data Registry, Google BigQuery에서 동일 조건 제공. 이미 수백 편 학술 논문에서 사용.

---

## GDELT 이벤트 구조

GDELT의 핵심은 (Actor1, Action, Actor2) 트리플이다. 61개 필드 중 6하원칙에 해당하는 것:

```
WHO:    Actor1Name, Actor1CountryCode, Actor1Type1Code
WHOM:   Actor2Name, Actor2CountryCode, Actor2Type1Code
WHAT:   EventCode (CAMEO 310개 이벤트 유형, 4단계 계층)
WHEN:   Day (YYYYMMDD), FractionDate
WHERE:  ActionGeo_Lat, ActionGeo_Long, ActionGeo_CountryCode, ActionGeo_FullName
HOW:    GoldsteinScale (-10 ~ +10 영향도), QuadClass (협력/갈등 4분류)
```

### GDELT 원본 예시

```json
{
  "GlobalEventId": 1035443882,
  "Day": "20210323",
  "Actor1": {
    "Name": "UNITED STATES",
    "Code": "USA",
    "CountryCode": "USA",
    "Type1Code": "GOV",
    "Geo_Fullname": "Hollywood, California, United States",
    "Geo_Lat": 34.0983,
    "Geo_Long": -118.327
  },
  "Actor2": {
    "Name": "SCIENTIST",
    "Code": "",
    "Type1Code": "SCI"
  },
  "EventCode": "020",
  "EventBaseCode": "020",
  "EventRootCode": "02",
  "QuadClass": 1,
  "GoldsteinScale": 3.0,
  "NumMentions": 4,
  "AvgTone": -5.17,
  "SOURCEURL": "https://www.wbur.org/..."
}
```

### 우리가 변환할 것

```
GDELT 원본                          →  SIDX 이벤트
─────────────────────────────────────────────────────────
Actor1Name: "BARACK OBAMA"          →  who: Q76 (Entity SIDX)
EventCode: "043" (Host a visit)     →  did: visit.v.01 (16비트)
Actor2Name: "CHINA"                 →  whom: Q148 (Entity SIDX)
Day: "20240315"                     →  when: 비트 인코딩
ActionGeo: lat/long/country         →  where: Q-ID (Entity SIDX)
GoldsteinScale: 3.0                 →  intensity: 양자화
```

---

## 매핑 작업

### 1. CAMEO EventCode → 워드넷 synset (수동, 1일)

CAMEO은 310개 이벤트 유형의 4단계 계층이다.

```
Root (20개):
  01 Make public statement
  02 Appeal
  03 Express intent to cooperate
  04 Consult
  05 Engage in diplomatic cooperation
  ...
  17 Coerce
  18 Assault
  19 Fight
  20 Use unconventional mass violence

→ 워드넷 매핑 예시:
  "020" (Appeal)           → appeal.v.01
  "043" (Host a visit)     → visit.v.01
  "050" (Diplomatic coop)  → cooperate.v.01
  "190" (Military force)   → attack.v.01
```

20개 루트 + 310개 리프. 수동 매핑 가능. 하루 작업. 매핑 테이블 JSON으로 공개.

### 2. Actor → 위키데이터 Q-ID 링킹 (부분 자동)

**국가 코드 (쉬움):**

```
GDELT 261개 FIPS 국가코드 → 위키데이터 Q-ID
"USA" → Q30, "CHN" → Q148, "KOR" → Q884, "JPN" → Q17
수동 매핑. 261개. 반나절.
```

**조직/기관 (중간):**

```
GDELT KnownGroupCode: "IGO" (정부간기구), "NGO", "MIL" 등
주요 기관: "UN" → Q1065, "NATO" → Q7184, "WHO" → Q7817
빈도 상위 500개 수동 매핑 + 나머지 Entity Linking.
```

**인물 (어려움):**

```
GDELT Actor1Name: "OBAMA" → Q76 (명확)
GDELT Actor1Name: "PARK"  → Q? (모호)

→ Entity Linking 모델 (REL / BLINK / mGENRE) 필요
→ 정확도 70-80% 예상
→ confidence 필드로 품질 관리
```

### 3. 매핑 품질 전략

```
Tier 1: 국가 (261개, 수동, 정확도 ~100%)
Tier 2: 주요 기관/조직 (500개, 수동, 정확도 ~100%)
Tier 3: 고빈도 인물 (10,000명, Entity Linking + 수동 검증, 정확도 ~95%)
Tier 4: 나머지 인물 (자동 Entity Linking, 정확도 ~70-80%)
```

PoC에서는 Tier 1-2만으로도 충분한 데모가 된다. 국가 간 이벤트 검색은 인물 링킹 없이도 작동한다.

---

## SIDX 이벤트 인코딩

### 이벤트 SIDX 구조

```
who:        64비트 (Entity SIDX — [type 6|subtype 5|region 4|era 3|미사용 14|Q-ID 32])
did:        16비트 (워드넷 동사 코드)
whom:       64비트 (Entity SIDX)
where:      64비트 (Entity SIDX — type=location)
when:       32비트 (연도 12b + 월 4b + 일 5b + 시 5b + 예약 6b)
intensity:  16비트 (GoldsteinScale 양자화 8b + QuadClass 2b + 예약 6b)
────────────────────────────────
합계:       256비트 = 32바이트/이벤트
```

### 용량 비교

| 방식 | 10억 이벤트 |
|------|-------------|
| GDELT 원본 CSV | ~2.5TB (연간 기준 추정) |
| RAG 임베딩 (768dim FP32) | ~3TB |
| SIDX 이벤트 (32바이트) | **~32GB** |

### 메모리 요구사항

```
Entity SIDX:    108.8M × 8B = ~830MB
Event SIDX:     10억 × 32B = ~32GB
코드북:          ~3MB
합계:           ~33GB

→ 64GB RAM 서버면 전부 메모리 상주 가능
→ 노트북에서 1억 이벤트 데모 = 3.2GB (충분)
```

---

## 데이터 수집: BigQuery

GDELT 전체가 Google BigQuery에 퍼블릭 데이터셋으로 있다. SQL 한 줄로 가져온다.

### 이벤트 추출 쿼리

```sql
SELECT
  GlobalEventId,
  Day,
  Actor1Name, Actor1CountryCode, Actor1Type1Code,
  Actor2Name, Actor2CountryCode, Actor2Type1Code,
  EventCode, EventBaseCode, EventRootCode,
  QuadClass, GoldsteinScale,
  ActionGeo_CountryCode, ActionGeo_Lat, ActionGeo_Long,
  ActionGeo_FullName,
  NumMentions, AvgTone,
  SOURCEURL
FROM `gdelt-bq.gdeltv2.events`
WHERE Year >= 2020
  AND NumMentions >= 3
ORDER BY Day DESC
```

`NumMentions >= 3`으로 노이즈 필터링. 3회 이상 언급된 이벤트만.

### 비용

```
BigQuery 무료 티어: 월 1TB 쿼리 무료
GDELT events 테이블: ~수백 GB
필요한 컬럼만 SELECT하면 스캔량 감소
→ 무료 티어 안에서 충분히 가능
```

### 규모 추정

```
GDELT 2.0 (2015~현재): ~수십억 이벤트
NumMentions >= 3 필터: ~수억 건 (추정)
NumMentions >= 5 필터: ~수천만 건 (추정)

PoC: NumMentions >= 5, Year >= 2020 → 수천만 건
풀 데모: NumMentions >= 3, 전 기간 → 수억~10억 건
```

---

## 검색 파이프라인

### 파이프라인 4단계

```
1단계: 쿼리 의미 추출        (코드북 룩업 / LLM)
2단계: 비트마스크 조립        (결정적 — 알고리즘)
3단계: NumPy 비트 AND 스캔   (결정적)
4단계: LLM 최종 판정         (소수 후보만)
```

### 엔티티 검색 (기존)

```
"한국 정치인"
  → mask/pattern 조립: type=person, subtype=politician, region=east_asia
  → results = index[(index & mask) == pattern]
  → 수백 개, ~20ms (NumPy)
```

### 이벤트 검색 (GDELT 기반)

```
"2024년에 미국이 중국에 대해 취한 외교 행동"
  → 조건:
      who.region = north_america (+ Q-ID로 USA 특정)
      whom.region = east_asia (+ Q-ID로 China 특정)
      did ∈ {diplomatic cooperation 계열 CAMEO}
      when.year = 2024

  → NumPy 비트마스크 스캔 (이벤트 SIDX 배열)
  → 수천 개, ~100ms
    → LLM 최종 판정 → 결과
```

### 관계 검색

```
"오바마와 시진핑이 만난 적 있어?"
  → who Q-ID = Q76 (Obama) AND whom Q-ID = Q15031 (Xi Jinping)
    OR who = Q15031 AND whom = Q76
  → did = meet/visit/consult 계열
  → NumPy 스캔 → 결과 이벤트 목록
```

```
"한일 외교 갈등 2020년 이후"
  → who.region ∈ {east_asia} + country Q-ID 필터
  → whom.region ∈ {east_asia} + country Q-ID 필터
  → QuadClass ∈ {3: Verbal Conflict, 4: Material Conflict}
  → when >= 2020
  → NumPy 스캔 → LLM 판정
```

---

## 성능 예산

### 엔티티 검색 (108.8M)

```
SIDX:          ~830MB
NumPy 비트 AND: ~20ms
LLM 판정:      ~50ms (소수 후보만)
합계:          ~70ms
```

### 이벤트 검색 (10억건)

```
SIDX:          ~32GB
NumPy 비트 AND: ~100ms
LLM 판정:      ~150ms (소수 후보만)
합계:          ~250ms
```

### 풀 RAG 대비

| 항목 | 풀 RAG (10억건) | SILK (NumPy) |
|------|-----------------|------------|
| 저장 | ~3TB+ | ~33GB |
| 검색 시간 | 5~30초 | ~250ms |
| RAM 요구 | 수 TB | 64GB |
| GPU 의존 | 전 구간 | LLM 판정만 |
| 월 비용 | $10,000+ | $100-300 |
| 재현성 | 비결정적 | 비트 AND까지 결정적 |

---

## 데모 시나리오 10개

### 엔티티 검색 (정적 지식)

| # | 쿼리 | SIDX 조건 |
|---|------|-----------|
| 1 | 1950년대 한국 남성 정치인 | Human, country=KR, era=1950s, gender=M, occ=politician |
| 2 | 오리온자리 G형 항성 | Star, constellation=Orion, spectral=G |
| 3 | 포유류 멸종위기 초식동물 | Taxon, class=Mammalia, conservation=endangered, diet=herb |

### 이벤트 검색 (동적 지식)

| # | 쿼리 | SIDX 조건 |
|---|------|-----------|
| 4 | 2024년 미국의 대중국 외교 행동 | who.country=USA, whom.country=CHN, when=2024 |
| 5 | 일론 머스크 관련 이벤트 2023년 | who=Q317521, when=2023 |
| 6 | 한일 외교 갈등 2020년 이후 | who/whom.country∈{KR,JP}, QuadClass∈{3,4}, when>=2020 |
| 7 | 러시아-우크라이나 군사 이벤트 2022년 | who/whom.country∈{RU,UA}, CAMEO root=18,19,20, when=2022 |

### 관계 검색 (엔티티 + 이벤트 결합)

| # | 쿼리 | 방법 |
|---|------|------|
| 8 | 오바마와 시진핑 회동 이력 | who/whom ∈ {Q76, Q15031}, did=meet계열 |
| 9 | 삼성전자 관련 국제 이벤트 | who/whom 이름 매칭 "SAMSUNG" |
| 10 | G7 국가 간 협력 이벤트 2023-2025 | who/whom.country ∈ G7, QuadClass∈{1,2}, when∈2023-2025 |

---

## 구현 순서

```
Week 1:  GDELT → BigQuery 쿼리 → 이벤트 추출 (수천만~수억 건)
         CAMEO 310개 → 워드넷 synset 매핑 테이블 작성
         국가 261개 FIPS → Q-ID 매핑 테이블 작성

Week 2:  Entity SIDX 분류율 62% → 90%+ (매핑 보강 + P279)
         이벤트 SIDX 인코딩 파이프라인 구축
         이벤트 데이터 SIDX 인코딩 실행

Week 3:  SIMD 벤치마크 (엔티티 5 시나리오 + 이벤트 5 시나리오)
         RAG 재정렬 붙이기
         풀 RAG 대비 벤치마크 실행

Week 4:  논문 작성
         GitHub 레포 정리 (README, 재현 스크립트, 데모 노트북)
         매핑 테이블 전부 공개
```

4주. 논문 + 데모 + GitHub 동시 출시.

---

## 논문 구조

```
제목:  "SILK: Symbolic Index for LLM Knowledge —
        Sub-Second Semantic Search over Billion-Scale
        Event Databases without Vector Embeddings"

Abstract:
  위키데이터 108.8M 엔티티 + GDELT N억 이벤트,
  33GB 저장, 250ms 검색, 풀 RAG 대비 20-100x 빠름.

Section 1: Introduction
  RAG의 한계 (비용, 속도, 정확도, 재현성)

Section 2: SILK Architecture
  뉴로-심볼릭 구조, SIDX 64비트 레이아웃, VALID 검증

Section 3: SIDX Encoding
  64비트 의미정렬 식별자, 타입별 독립 스키마, 우아한 열화

Section 4: Event SIDX
  GDELT → SIDX 변환 파이프라인
  CAMEO → 워드넷 매핑, Actor → Q-ID 링킹

Section 5: SIMD Bitwise Search
  비트마스크 조립, 전수 스캔, 선택도 분석

Section 6: Hybrid SIMD + LLM
  4단계 파이프라인, 느슨 SIMD + LLM 재정렬 전략

Section 7: Benchmarks
  10 시나리오 (엔티티 3 + 이벤트 4 + 관계 3)
  풀 RAG 대비: 속도, 저장, 정확도, 재현성, 비용

Section 8: Discussion
  코드북 커버리지 한계, GDELT 데이터 품질 (정확도 55%, 중복 20%)
  느슨 SIMD 전략으로 거짓 음성 완화

Section 9: Conclusion + GitHub link

인용:
  GDELT Project (Leetaru & Schrodt, 2013)
  위키데이터 (Vrandečić & Krötzsch, 2014)
  워드넷 (Miller, 1995)
```

---

## GitHub 레포 구조

```
silk/
├── README.md              # 프로젝트 개요 + 벤치마크 숫자
├── LICENSE                # MIT 또는 Apache 2.0
├── paper/
│   └── silk-paper.pdf     # arXiv 논문
├── data/
│   ├── entity_types_64.json
│   ├── type_schemas.json
│   ├── codebooks_full.json
│   ├── cameo_to_wordnet.json      # CAMEO → 워드넷 매핑
│   ├── fips_to_wikidata.json      # FIPS 국가 → Q-ID 매핑
│   └── README.md                  # 데이터 설명
├── silk/
│   ├── codebook.py                # 코드북 로드, lookup
│   ├── encoder.py                 # tag → SIDX uint64
│   ├── valid.py                   # VALID 검증
│   ├── search.py                  # NumPy 비트 AND 검색
│   └── query.py                   # 쿼리 → mask/pattern
├── scripts/
│   └── gdelt_pipeline.py          # BigQuery → SIDX 파이프라인
├── benchmarks/
│   ├── scenarios.json             # 10개 시나리오 정의
│   ├── run_benchmarks.py          # 벤치마크 실행 스크립트
│   ├── run_fullrag_baseline.py    # 풀 RAG 베이스라인
│   └── results/                   # 결과 JSON + 차트
├── notebooks/
│   ├── 01_gdelt_to_sidx.ipynb     # GDELT 데이터 변환 튜토리얼
│   ├── 02_entity_search.ipynb     # 엔티티 검색 데모
│   ├── 03_event_search.ipynb      # 이벤트 검색 데모
│   └── 04_benchmarks.ipynb        # 벤치마크 시각화
└── demo/
    └── app.py                     # Streamlit 라이브 데모 (선택)
```

---

## 공개 전략

### HN/Reddit 제목

```
"SILK: I indexed 1B news events in 33GB and search them in 250ms
 — no embeddings, no vector DB, just 64-bit semantic IDs + SIMD"
```

또는:

```
"Why I replaced my 3TB vector database with 33GB of bitmasks
 — and got better accuracy"
```

### SILK 로드맵

```
v1.0:  SILK (SIDX + SIMD + LLM 검수). 뉴로-심볼릭 검색.
       "64비트 의미정렬 ID"로만 설명.

v1.5:  이벤트 간 관계 표현 추가.
       GEUL 문법 도입.

v2.0:  VALID 파이프라인 강화.
       GEUL 프로젝트 공식 소개.

v3.0:  SILK-native LLM 통합.
       전 구간 구조화.
```

v1.0에서 SILK만으로 충분히 유용하다.
한계를 느낀 사람이 GEUL을 찾아온다.

---

## GDELT 데이터 품질 대응

GDELT 핵심 필드 정확도 ~55%, 중복률 ~20%라는 연구가 있다.

### 대응 전략

```
중복 제거:    GlobalEventId + Day + Actor1 + Actor2 + EventCode 조합으로 dedup
정확도 보완:  NumMentions >= 5 필터 (다수 매체 언급 = 높은 신뢰도)
              confidence 필드 활용
Q-ID 링킹:   링킹 실패 시 country-level로 폴백 (Tier 1 매핑)
```

### 논문에서의 프레이밍

"GDELT 원본 정확도 ~55% → SIDX 인코딩 + Q-ID 링킹 + 중복 제거로 보정"
데이터 품질 개선도 기여의 일부로 제시.

---

## 정리

GDELT가 46년간 뉴스 이벤트를 구조화해놨다.
우리는 코드 체계만 변환하면 된다. CAMEO → 워드넷, Actor → Q-ID.

위키데이터로 "누가 무엇인지"를 알고,
GDELT로 "누가 무엇을 했는지"를 안다.
33GB에 담고 250ms에 답한다.

4주면 논문 + 데모 + GitHub이 동시에 나온다.
SILK로 출발한다. 한계를 느끼면 그때 GEUL로.
