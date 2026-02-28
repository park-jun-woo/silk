# SILK 검수 파이프라인

*2026년 2월 28일*

SILK 아키텍처의 3단계 품질 보증 체계: 소형 LLM 태깅 → VALID 기계 검사 → 대형 LLM 검수.

---

## 핵심 명제

```
벡터 임베딩: 블랙박스 → 검수 불가 → 품질 기도
SILK SIDX:   화이트박스 → JSON 검수 → 품질 보증

SIDX는 JSON으로 읽힌다.
사람도 읽을 수 있고, LLM도 읽을 수 있다.
읽을 수 있으니까 검수할 수 있다.
검수할 수 있으니까 품질을 보증할 수 있다.
```

---

## 벡터 임베딩은 왜 검수가 불가능한가

```
소형 임베딩 모델이 "이순신"을 벡터로 변환:
  [0.234, -0.891, 0.445, 0.012, ..., 0.112]  (384차원 float)

검수하려면:
  "0.234가 맞는가?" → 의미 없는 질문.
  "127번째 차원이 정확한가?" → 판단 기준 없음.
  "이 벡터를 다른 LLM에게 보여주고 맞는지 물어보자"
  → LLM이 384개 float를 읽고 의미를 판단? 불가능.

결과:
  임베딩 모델이 뱉은 벡터를 그대로 믿을 수밖에 없음.
  오염된 벡터가 인덱스에 들어가도 알 방법 없음.
  품질 관리 = 기도.
```

## SILK는 왜 검수가 가능한가

```
소형 LLM이 "이순신"을 태깅:
  {"type": "person", "subtype": "military", "region": "east_asia",
   "era": "early_modern", "qid": "Q484523"}

검수하려면:
  "type이 person 맞아?" → 맞음.
  "subtype이 military 맞아?" → 맞음. 무신이니까.
  "era가 early_modern 맞아?" → 맞음. 1545-1598이니까.
  → 사람도 판단 가능. LLM도 판단 가능. 자연어니까.

결과:
  모든 필드를 독립적으로 검증 가능.
  잘못된 필드만 골라서 수정 가능.
  품질 관리 = 시스템.
```

---

## 3단계 검수 파이프라인

```
[1단계] 소형 LLM 대량 태깅
    ↓
[2단계] VALID 기계 검사 ──FAIL──→ 자동 재시도 (최대 3회)
    │                                    │
   PASS                          3회 실패 → 사람 검토 큐
    │
    ▼
[3단계] 대형 LLM 검수 (전수 / 샘플 / 스마트)
    │
   PASS ──→ 비트 조립 → SIDX → 인덱스
    │
   FAIL ──→ 대형 LLM이 수정값 제시 → VALID 재검사 → 인덱스
```

---

## 1단계: 소형 LLM 대량 태깅

### 모델 선택

```
| 모델 | 속도 | 비용/건 | 초회 정확도 | 용도 |
|------|------|--------|-----------|------|
| Llama 8B (로컬) | 1만건/분 | $0 | 82-88% | 대량 처리, 비용 최소 |
| GPT-4o-mini | 5천건/분 | $0.00003 | 88-92% | 가성비 최적 |
| Claude Haiku | 5천건/분 | $0.00005 | 90-93% | 정확도 우선 |
```

### 태깅 프롬프트

```
[시스템]
당신은 문서 분류 전문가입니다.
다음 문서를 읽고, 각 필드에 대해 주어진 유효값 목록에서
정확히 하나만 선택하여 태그하십시오.

규칙:
- 반드시 유효값 내에서만 선택할 것.
- 적합한 값이 없으면 "other" 또는 "unknown"을 선택할 것.
- 각 필드에 대한 확신도를 high/medium/low로 표시할 것.
- JSON만 출력. 다른 텍스트 없이.

[필드 정의]
type (필수): person, organization, location, creative_work, event,
             taxon, star, galaxy, chemical, gene, painting, ...
subtype (필수): {type에 따른 유효값 목록 — 아래 참조}
  type=person: politician, scientist, artist, athlete, writer,
               military, religious, business, diplomat, activist,
               educator, engineer, medical, legal, entertainer, ...
  type=organization: company, university, ngo, government,
                     military, religious, media, sports, ...
  type=location: city, country, mountain, river, building, ...
  ...
region (필수): east_asia, southeast_asia, south_asia, central_asia,
               west_asia, north_africa, sub_saharan, west_europe,
               east_europe, north_europe, south_europe, north_america,
               central_america, south_america, oceania, global
era (필수): ancient, medieval, early_modern, modern, contemporary,
            current, timeless, unknown

[출력 형식]
{
  "type": "...",         "type_conf": "high|medium|low",
  "subtype": "...",      "subtype_conf": "high|medium|low",
  "region": "...",       "region_conf": "high|medium|low",
  "era": "...",          "era_conf": "high|medium|low",
  "qid": "Q..." or null
}

[문서]
{document_text}
```

### 태깅 결과 예시

```
문서: "이순신(1545-1598)은 조선 중기의 무신이자 발명가다.
       거북선을 설계하고 임진왜란에서 조선 수군을 이끌었다."

소형 LLM 응답:
{
  "type": "person",          "type_conf": "high",
  "subtype": "military",     "subtype_conf": "high",
  "region": "east_asia",     "region_conf": "high",
  "era": "early_modern",     "era_conf": "medium",
  "qid": "Q484523"
}
```

### Multi-SIDX 태깅 프롬프트

```
[시스템 — 문서 전체에서 모든 엔티티/이벤트 추출]

다음 문서를 분석하여 언급된 모든 엔티티와 이벤트를 추출하시오.
각 항목에 대해 type/subtype/region/era를 태그하고 확신도를 표시할 것.
type 필드가 엔티티(person, org 등)와 이벤트(event)를 구분한다.

[문서]
"조선일보 2026.02.27 — 삼성전자의 이재용 회장,
 엔비디아의 젠슨황 CEO, 현대자동차의 정의선 회장이
 서울 깐부치킨에서 회동했다."

[소형 LLM 응답]
{
  "entities": [
    {
      "type": "person", "subtype": "business",
      "region": "east_asia", "era": "current",
      "entity": "이재용", "qid": "Q484523",
      "conf": "high"
    },
    {
      "type": "organization", "subtype": "company",
      "region": "east_asia", "era": "current",
      "entity": "삼성전자", "qid": "Q81965",
      "conf": "high"
    },
    {
      "type": "person", "subtype": "business",
      "region": "north_america", "era": "current",
      "entity": "젠슨황", "qid": "Q7352287",
      "conf": "high"
    },
    {
      "type": "organization", "subtype": "company",
      "region": "north_america", "era": "current",
      "entity": "엔비디아", "qid": "Q182477",
      "conf": "high"
    },
    {
      "type": "person", "subtype": "business",
      "region": "east_asia", "era": "current",
      "entity": "정의선", "qid": "Q20895436",
      "conf": "high"
    },
    {
      "type": "organization", "subtype": "company",
      "region": "east_asia", "era": "current",
      "entity": "현대자동차", "qid": "Q55043",
      "conf": "high"
    },
    {
      "type": "event", "subtype": "meeting",
      "region": "east_asia", "era": "current",
      "entity": "회동", "qid": null,
      "conf": "high"
    }
  ]
}
```

### 배치 처리

```
1.08억건 기준:

GPT-4o-mini:
  병렬 워커 100개 × 50건/초 = 5000건/초
  108,000,000 / 5000 = 21,600초 = 6시간
  비용: 108M × $0.00003 = $3,240

로컬 Llama 8B (A100 × 4):
  4 GPU × 2500건/분 = 10,000건/분
  108,000,000 / 10,000 = 10,800분 = 7.5일
  비용: $0 (전기세 ~$200)
```

---

## 2단계: VALID 기계 검사

### 검사 레벨

#### 레벨 1 — 유효값 체크 (즉시)

```
모든 필드의 값이 코드북에 존재하는가?

입력: {"type": "person", "subtype": "admiral", "region": "east_asia"}

검사:
  type "person"     → ✅ codebook.type.values에 있음
  subtype "admiral"  → ❌ codebook.subtype.person에 없음
  region "east_asia" → ✅ codebook.region.values에 있음

결과: REJECT
사유: subtype "admiral" 유효하지 않음
```

```python
def validate_field(value: str, valid_values: list[str]) -> str | None:
    if value not in valid_values:
        return f"invalid value '{value}', valid: {valid_values}"
    return None
```

#### 레벨 2 — 타입 정합성 체크 (즉시)

```
type과 subtype 조합이 허용되는가?

입력: {"type": "painting", "subtype": "politician"}

검사:
  painting의 유효 subtype: [oil, watercolor, portrait, landscape, ...]
  "politician"은 painting의 subtype이 아님.

결과: REJECT
사유: type-subtype 불일치
```

```python
def validate_consistency(tag: dict, codebook: dict) -> str | None:
    valid_subs = codebook["subtype"]["values_per_type"][tag["type"]]
    if tag["subtype"] not in valid_subs:
        return f"subtype '{tag['subtype']}' invalid for type '{tag['type']}', valid: {valid_subs}"
    return None
```

#### 레벨 3 — 제약조건 체크 (즉시)

```
sidx.yaml에 정의된 제약조건 위반 여부.

입력: {"type": "star", "region": "east_asia", "era": "modern"}

제약조건 (sidx.yaml):
  - if: { type: star }
    then: { region: global, era: timeless }

검사:
  type=star인데 region=east_asia → 위반
  type=star인데 era=modern → 위반

결과: AUTO-CORRECT
  region: "east_asia" → "global"
  era: "modern" → "timeless"
  로그: "constraint auto-correction applied"
```

```python
def apply_constraints(tag: dict, constraints: list[dict]) -> list[str]:
    corrections = []
    for c in constraints:
        if all(tag.get(k) == v for k, v in c["if"].items()):
            for k, v in c["then"].items():
                if tag.get(k) != v:
                    corrections.append(f"{k}: '{tag[k]}' → '{v}'")
                    tag[k] = v
    return corrections
```

#### 레벨 4 — JSON 구조 체크 (즉시)

```
파싱 가능한 JSON인가? 필수 필드가 모두 있는가?

입력: "이순신은 군인입니다"  (JSON이 아닌 자연어)
결과: REJECT — JSON 파싱 실패

입력: {"type": "person"}     (필수 필드 누락)
결과: REJECT — subtype, region, era 누락
```

### 재시도 로직

```
VALID 실패 시 재시도 프롬프트:

[재시도 1회차]
"이전 응답이 유효하지 않습니다.
 사유: subtype "admiral"은 유효값이 아닙니다.
 type=person의 유효 subtype: [politician, scientist, artist,
   athlete, writer, military, religious, business, diplomat, ...]
 다시 태그하십시오. JSON만 출력."

[재시도 2회차]
"다시 유효하지 않습니다.
 사유: {구체적 사유}
 유효값 목록을 정확히 따르십시오.
 {유효값 목록 전체 재제시}"

[재시도 3회차 실패]
→ 사람 검토 큐로 이동.
→ doc_id, 문서 원문, 실패 사유, LLM 응답 3회분 기록.
```

### 재시도 통계

```
1.08억건 태깅 시 예상:

초회 VALID 통과:    ~90%     = 9720만건 → 인덱스
1회 재시도 통과:    ~7%      = 756만건 → 인덱스
2회 재시도 통과:    ~2%      = 216만건 → 인덱스
3회 재시도 통과:    ~0.5%    = 54만건 → 인덱스
3회 실패:          ~0.5%    = 54만건 → 사람 검토 큐

자동 처리율: 99.5%
인덱스 오염: 0건 (VALID 통과 못하면 인덱스에 못 들어감)
```

---

## 3단계: 대형 LLM 검수

### 왜 가능한가

```
SIDX의 JSON 표현:
{
  "type": "person",
  "subtype": "military",
  "region": "east_asia",
  "era": "early_modern",
  "qid": "Q484523"
}

이것은:
  사람이 읽을 수 있다.     → 사람 검수 가능.
  소형 LLM이 생성했다.     → LLM 태스크.
  대형 LLM이 읽을 수 있다. → LLM 검수 가능.

벡터 [0.234, -0.891, ...]:
  사람이 읽을 수 없다.     → 사람 검수 불가능.
  임베딩 모델이 생성했다.   → 블랙박스.
  대형 LLM이 읽을 수 없다. → LLM 검수 불가능.
```

### 검수 프롬프트

```
[시스템]
당신은 SIDX 태그 품질 검수 전문가입니다.
다음 문서 원문과 SIDX 태그를 비교하여 각 필드의 정확성을 판정하십시오.

판정 기준:
- PASS: 정확한 태깅
- WARN: 허용 가능하나 더 나은 값이 있음 (수정값 제시)
- FAIL: 명백한 오류 (수정값 제시)

[문서 원문]
"마리 퀴리(1867-1934)는 폴란드 출생의 프랑스 물리학자이자
 화학자다. 노벨상을 두 번 수상한 유일한 인물이다."

[SIDX 태그]
{
  "type": "person",
  "subtype": "scientist",
  "region": "west_europe",
  "era": "modern",
  "qid": "Q7186"
}

[검수 응답]
{
  "type":    {"verdict": "PASS", "note": "인물 맞음"},
  "subtype": {"verdict": "PASS", "note": "물리학자+화학자, scientist 적합"},
  "region":  {"verdict": "WARN",
              "note": "폴란드 출생이나 주요 활동지가 프랑스. west_europe 허용 가능하나 east_europe도 고려",
              "suggestion": "Multi-SIDX로 east_europe 추가 권장"},
  "era":     {"verdict": "PASS", "note": "1867-1934 = modern(1800-1945) 정확"},
  "qid":     {"verdict": "PASS", "note": "Q7186 = Marie Curie 확인"},
  "overall": "PASS with WARN",
  "recommendations": [
    "region=east_europe 추가 SIDX 엔트리 생성 권장 (폴란드 출생)"
  ]
}
```

### 검수 전략

#### 전략 A — 전수 검수

```
모든 SIDX 태그를 대형 LLM이 검수.

대상: 1.08억건 전체
비용: 108M × $0.003 = $324,000
시간: ~24시간 (대규모 병렬)
정확도: ~99.5%

용도: 최고 품질이 필요한 경우 (의료, 법률, 금융)
```

#### 전략 B — 샘플 검수

```
랜덤 샘플링으로 전체 품질 추정.

대상: 전체의 5% = 540만건
비용: 5.4M × $0.003 = $16,200
시간: ~2시간
정확도 추정: ±0.5% (95% 신뢰구간)

방법:
  540만건 검수 → 오류율 2.1% 측정
  → "전체 오류율 1.6%~2.6% (95% CI)"
  → 허용 범위 내 → 배포

용도: 일반적인 경우. 가성비 최적.
```

#### 전략 C — 스마트 검수 (권장)

```
소형 LLM의 confidence 기반 선별 검수.

원리:
  소형 LLM이 "자신 없는" 태그만 대형 LLM에게 넘김.
  confidence=high → 검수 스킵
  confidence=medium → 해당 필드만 검수
  confidence=low → 전체 재태깅

분포 예상 (1.08억건):
  전 필드 high:   65%  = 7020만건 → 스킵
  1-2필드 medium: 25%  = 2700만건 → 필드 단위 검수
  1+필드 low:      8%  = 864만건 → 전체 검수
  VALID 실패 후:   2%  = 216만건 → 재태깅 + 검수

비용:
  스킵:        $0
  필드 검수:   2700만 × $0.001 = $27,000
  전체 검수:   864만 × $0.003  = $25,920
  재태깅 검수: 216만 × $0.003  = $6,480

  총: $59,400

전수($324,000) 대비 82% 절약.
샘플($16,200)보다는 비싸지만 실제 오류를 잡아냄.
```

### 스마트 검수 상세 흐름

```
소형 LLM 태깅 결과:
{
  "type": "person",         "type_conf": "high",
  "subtype": "military",    "subtype_conf": "high",
  "region": "east_asia",    "region_conf": "high",
  "era": "early_modern",    "era_conf": "medium",   ← 이것만 검수
  "qid": "Q484523"
}

대형 LLM 검수 프롬프트 (필드 단위):
  "다음 문서에서 시대(era)가 'early_modern(1500-1800)'이 맞습니까?
   문서: '이순신(1545-1598)은 조선 중기의 무신이다.'
   PASS 또는 FAIL + 올바른 값을 답하시오."

대형 LLM 응답:
  {"era_verdict": "PASS", "note": "1545-1598은 early_modern 범위 내"}

→ 필드 하나만 검수. 전체 재태깅 아님.
→ 비용: 전체 검수의 1/3.
```

### 검수 결과 처리

```
PASS → 그대로 비트 조립 → 인덱스

WARN → 기록 + 그대로 비트 조립
  → Multi-SIDX 추가 엔트리 생성 권장 목록에 추가
  → 배치로 처리

FAIL → 대형 LLM의 수정값 적용
  → 수정값 VALID 재검사 → PASS → 비트 조립
  → 수정값도 VALID 실패 → 사람 검토 큐
```

---

## LLM 환각 패턴별 방어

### 패턴 1 — 없는 키워드 환각

```
소형 LLM: {"subtype": "admiral"}
VALID: ❌ 코드북에 "admiral" 없음
재시도: "유효값: [politician, scientist, military, ...]"
소형 LLM: {"subtype": "military"} → ✅

방어 주체: VALID (2단계)
대형 LLM 개입: 불필요
```

### 패턴 2 — 타입 혼동

```
소형 LLM: {"type": "person", "subtype": "oil_painting"}
VALID: ❌ person의 subtype에 "oil_painting" 없음
재시도: person 유효 subtype 목록 제시
소형 LLM: {"subtype": "artist"} → ✅

방어 주체: VALID (2단계)
대형 LLM 개입: 불필요
```

### 패턴 3 — 과잉 구체화

```
소형 LLM: {"subtype": "16th_century_joseon_naval_commander"}
VALID: ❌ 코드북에 없음
재시도: 유효값 목록 제시
소형 LLM: {"subtype": "military"} → ✅

방어 주체: VALID (2단계)
참고: 세밀한 분류는 코드북이 아니라 Q-ID가 담당
```

### 패턴 4 — 애매한 분류 (VALID 통과하지만 부정확)

```
소형 LLM: {"type": "person", "subtype": "artist"}  (이순신을 artist로)
VALID: ✅ person/artist는 유효한 조합
문제: subtype이 맞지 않지만 VALID는 통과

→ 이것이 대형 LLM 검수가 필요한 이유.

대형 LLM 검수:
  "이순신을 artist로 태깅. 문서에 '무신'이라 되어있음.
   FAIL. 수정: military"

방어 주체: 대형 LLM (3단계)
VALID만으로는 잡을 수 없는 "의미적 오류".
```

### 패턴 5 — Q-ID 오매칭

```
소형 LLM: {"entity": "이순신", "qid": "Q123456"}  (엉뚱한 Q-ID)
VALID: Q-ID 형식은 맞으니까 통과할 수 있음.

→ 위키데이터 lookup으로 추가 검증:
  Q123456 → "List of airports in Brazil"
  "이순신" ≠ "브라질 공항 목록"

→ Q-ID - entity name 불일치 → REJECT

방어 주체: Q-ID 검증 로직 (2.5단계)
```

```python
def validate_qid(tag: dict, wikidata: dict) -> str | None:
    qid = tag.get("qid")
    if not qid:
        return None  # Q-ID 없는 엔티티 허용
    label = wikidata.get(qid)
    if not label:
        return f"Q-ID {qid} does not exist"
    # 선택: entity명과 label 유사도 체크
    # (완전 일치 아니어도 됨 — 이순신 vs Yi Sun-sin)
    return None
```

### 패턴 6 — JSON 구조 깨짐

```
소형 LLM: "이순신은 조선의 유명한 장군입니다."  (JSON 아닌 자연어)
VALID: ❌ JSON 파싱 실패
재시도: "JSON만 출력하시오. 형식: {\"type\": \"...\", ...}"

소형 LLM: {"type": "person", "subtype": "military" (닫는 괄호 누락)
VALID: ❌ JSON 파싱 실패
재시도: "유효한 JSON을 출력하시오."

방어 주체: JSON 파서 (2단계 이전, 사실상 0단계)
```

### 패턴 7 — 필드 누락

```
소형 LLM: {"type": "person", "subtype": "military"}
          (region, era 누락)
VALID: ❌ 필수 필드 누락
재시도: "모든 필드를 빠짐없이 채우시오: type, subtype, region, era"

방어 주체: VALID (2단계)
```

### 방어 요약

```
| 환각 패턴 | 방어 주체 | 잡히는 단계 |
|----------|----------|-----------|
| 없는 키워드 | VALID 유효값 체크 | 2단계 |
| 타입 혼동 | VALID 정합성 체크 | 2단계 |
| 과잉 구체화 | VALID 유효값 체크 | 2단계 |
| 애매한 분류 | 대형 LLM 의미 검수 | 3단계 |
| Q-ID 오매칭 | 위키데이터 lookup | 2.5단계 |
| JSON 깨짐 | JSON 파서 | 0단계 |
| 필드 누락 | VALID 필수 필드 체크 | 2단계 |

2단계(VALID)에서 7개 중 5개 자동 차단.
3단계(대형 LLM)에서 나머지 1개 검수.
0단계(JSON 파서)에서 1개 차단.

인덱스 도달 전 방어막: 3중.
인덱스 오염 가능성: 사실상 0.
```

---

## 비용 총정리

### 시나리오별

```
| 시나리오 | 1단계 태깅 | 2단계 VALID | 3단계 검수 | 총 비용 | 정확도 |
|---------|----------|-----------|----------|--------|-------|
| 최저가 | Llama 8B $0 | $0 | 없음 | ~$200 (전기) | ~90% |
| 가성비 | 4o-mini $3,240 | $0 | 샘플 5% $16,200 | $19,440 | ~97% |
| 균형 | 4o-mini $3,240 | $0 | 스마트 $59,400 | $62,640 | ~99% |
| 최고품질 | Haiku $5,400 | $0 | 전수 $324,000 | $329,400 | ~99.5% |
```

1.08억건 기준. 1회성 비용.

### 벡터 임베딩과 비교

```
벡터 DB (Pinecone):
  임베딩 생성: $3,000~50,000 (모델 크기 의존)
  벡터 저장: $3,000/월
  검수: 불가능. $0이 아니라 불가능.
  연간: $36,000+ (저장만)

SIDX (가성비 옵션):
  태깅 + 검수: $19,440 (1회)
  저장: 8GB. $0.50/월.
  검수: 가능. 필요할 때만.
  연간: $19,446 (첫해), $6/년 (이후)
```

---

## 지속적 품질 모니터링

### 배치 후 통계 분석

```
[레벨 4 통계적 이상치 탐지]

전체 태깅 완료 후 버킷 크기 분석:

정상: type=person, sub=politician, region=east_asia: 2,100건
이상: type=person, sub=politician, region=global: 450,000건
  → 기대치 10배 이상
  → 원인: 소형 LLM이 지역 판별 실패, "global"로 도피
  → 조치: region=global인 politician만 재태깅 (프롬프트에 지역 판별 규칙 강화)
```

```python
import numpy as np
from collections import Counter

def detect_anomalies(index: np.ndarray, total_buckets: int) -> list[dict]:
    # 상위 18비트로 버킷 추출
    buckets = index >> 46
    counts = Counter(buckets.tolist())
    expected = len(index) // total_buckets

    alerts = []
    for bucket, count in counts.items():
        ratio = count / max(expected, 1)
        if ratio > 10.0:
            alerts.append({
                "bucket": bucket,
                "count": count,
                "expected": expected,
                "ratio": ratio,
                "action": "re-tag this bucket",
            })
    return sorted(alerts, key=lambda a: a["ratio"], reverse=True)
```

### 사용자 피드백 루프

```
검색 결과에서 사용자가 "이 결과 잘못됨" 신고:

  → 해당 문서의 SIDX 태그 확인
  → 어떤 필드가 잘못됐는지 즉시 확인 가능 (화이트박스)
  → 수정 → 비트 업데이트 → 즉시 반영

벡터 DB에서 같은 상황:
  → "이 검색 결과 잘못됨"
  → 벡터의 어디가 잘못됐는지 확인 불가
  → 재임베딩? 모델 교체? 알 수 없음
```

---

## 설계 철학

```
SILK 검수 파이프라인의 본질:

  코드북이 문지기다.
  유효값 목록이 울타리다.
  LLM이 아무리 환각해도 울타리 바깥의 값은
  물리적으로 인덱스에 들어갈 수 없다.

  그리고 울타리 안의 값은 JSON으로 읽힌다.
  읽히니까 사람이 검수할 수 있다.
  읽히니까 대형 LLM이 검수할 수 있다.

  벡터는 울타리가 없다. 어떤 벡터든 들어간다.
  벡터는 읽히지 않는다. 누구도 검수할 수 없다.

  이것이 화이트박스와 블랙박스의 차이다.
  이것이 SILK와 벡터 임베딩의 근본적 차이다.
```
