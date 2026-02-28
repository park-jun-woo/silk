# SILK 쿼리 생성 전략

**작성일:** 2026-02-26
**주제:** SILK에서 SIDX 비트마스크 쿼리를 어떻게 생성할 것인가 — LLM vs 알고리즘
**결론:** 하이브리드. 비트 조립은 100% 알고리즘, 자연어 의미 파싱만 LLM 보조.

---

## 1. 핵심 질문

SILK에서 NumPy 비트 AND 탐색을 하려면, 자연어 프롬프트에서 SIDX 비트마스크를 만들어내야 한다.

- LLM 모델이 필요한가?
- 알고리즘으로 가능한가?

---

## 2. 결론: 2단계 분리

```
자연어 쿼리  →  [Stage 1: 의미 추출]  →  구조화된 쿼리  →  [Stage 2: 비트 조립]  →  SIDX 비트마스크
                   ↑ 쟁점                                    ↑ 100% 알고리즘
```

- **Stage 2 (비트 조립)**: 논쟁의 여지 없이 순수 알고리즘. 코드북이 확정되면 `type=person, subtype=politician, region=east_asia` → 비트 조립은 비트 연산 함수 하나면 끝.
- **Stage 1 (의미 추출)**: 80-90%는 코드북 사전 룩업 + 규칙 NLU. 나머지 10-20%만 LLM 보조.

---

## 3. 알고리즘으로 커버 가능한 범위 (~80-90%)

| 쿼리 유형 | 방법 | 예시 |
|-----------|------|------|
| 타입 직접 지정 | 사전 매핑 | "사람" → type=person |
| 하위분류 지정 | 코드북 룩업 | "정치인" → subtype=politician |
| 지역 지정 | 코드북 룩업 | "한국" → region=east_asia |
| 시대 지정 | 코드북 룩업 | "현대" → era=modern |
| Q-ID 직접 지정 | 위키데이터 | "이순신" → qid=Q484523 |

**SIDX의 4개 필드에 의미가 고정**되어 있으므로, 개념→비트 매핑은 유한한 사전(dictionary)이다.

### 3.1 엔티티 쿼리는 거의 100% 알고리즘

4개 필드(type/subtype/region/era)가 코드북에 확정되어 있으므로:

```python
# "한국 정치인" → 코드북 룩업으로 mask/pattern 즉시 생성
TYPE_DICT = {"person": 0, "organization": 1, "location": 2, "event": 4, ...}
SUBTYPE_DICT = {"person": {"politician": 0, "scientist": 1, "military": 5, ...}}
REGION_DICT = {"east_asia": 0, "north_america": 11, ...}

mask    = np.uint64((0x3F << 58) | (0x1F << 53) | (0x0F << 49))
pattern = np.uint64((0 << 58) | (0 << 53) | (0 << 49))
results = index[(index & mask) == pattern]
```

---

## 4. LLM이 필요한 영역 (~10-20%)

| 쿼리 유형 | 왜 LLM? | 예시 |
|-----------|---------|------|
| **모호성 해소** | 같은 단어가 여러 타입 | "Apple" → Business? Taxon? |
| **함축 추론** | 명시하지 않은 속성 추론 | "위대한 과학자" → 저명도=7? |
| **시대/맥락 추론** | 배경지식 필요 | "나폴레옹 시대 사람" → Era=Early Modern |
| **유사/비유 쿼리** | 직접 매핑 불가 | "물과 관련된 것" → Chemical, River, Lake... |
| **복합 서술 쿼리** | 문장 구조 파싱 | "누군가 어제 서울에서 책을 산 사건" |

---

## 5. 3계층 아키텍처 제안

```
Level 1: 코드북 직접 룩업 (O(1), 알고리즘)
   ↓ 실패 시
Level 2: 규칙 기반 NLU (spaCy + 패턴 매칭)
   ↓ 실패 시
Level 3: LLM 호출 (마지막 수단)
```

**Level 1 — 코드북 룩업:**
```python
TYPE_DICT    = {"사람": "person", "인간": "person", "조직": "organization", ...}
SUBTYPE_DICT = {"정치인": "politician", "과학자": "scientist", "군인": "military", ...}
REGION_DICT  = {"한국": "east_asia", "미국": "north_america", "프랑스": "west_europe", ...}
ERA_DICT     = {"고대": "ancient", "중세": "medieval", "현대": "modern", "현재": "current", ...}
```

**Level 2 — 규칙 NLU:**
spaCy 형태소 분석 → 명사구/동사구 추출 → Level 1 사전에 매핑.

**Level 3 — LLM:**
LLM은 SIDX 비트를 직접 생성하지 않는다. 구조화된 JSON까지만 출력하고, 비트 조립은 알고리즘이 담당.

---

## 6. LLM의 역할: 비트 생성이 아닌 의미 파싱

LLM에게 주는 건 **스키마가 고정된 JSON 템플릿**이다:

```json
// SIDX 쿼리 스키마 — 4개 필드가 전부
{
  "type": null,      // 64개 중 택1 (null = don't care)
  "subtype": null,   // 32개 중 택1 (type별 다름)
  "region": null,    // 16개 중 택1
  "era": null,       // 8개 중 택1
  "qid": null        // Q-ID 직접 지정 (선택)
}
```

LLM은 null을 채우는 것이 전부. 못 채우면 null로 둔다 — 그게 곧 우아한 열화이고, NumPy 비트 AND에서는 don't care 비트가 된다.

이 패턴은 현대 LLM의 **function calling / tool use**와 구조적으로 동일하다.

---

## 7. 우아한 열화가 쿼리 오류 허용도를 높인다

SIDX의 "비트를 덜 채울수록 추상적 표현" 원칙이 쿼리에도 적용된다:

```
"한국 사람"
→ pattern: type=person, region=east_asia
→ mask:    type 6비트 + region 4비트만 ON

subtype, era는 null → don't care 비트 → 더 넓은 결과
```

LLM이 확신 없는 필드는 빼면 된다. NumPy가 넓은 범위를 탐색하고 LLM이 후보를 좁힌다.

---

## 8. 과도기 JSON 중간층의 가치

지금 LLM에게 비트를 직접 다루라고 하면 정확도가 보장되지 않는다.
JSON 중간층은 비용이 아니라 투자다:

1. **검증 가능성** — JSON은 사람이 읽고 오류를 바로 잡을 수 있다
2. **디버깅** — 쿼리 오류의 원인을 JSON 단에서 찾을 수 있다
3. **코드북 진화** — 비트 스키마가 바뀌어도 JSON→비트 변환 함수만 고치면 된다
4. **학습 데이터 축적** — (자연어, JSON) 쌍이 쌓이면 미래 GEUL-native 모델 학습 데이터가 된다

---

## 9. Tool Use와 SILK 쿼리의 구조적 동일성

```
[현재 Tool Use]
사용자: "이 파일에서 TODO 찾아줘"
→ LLM 추론 → Grep(pattern="TODO", path="./src") → 런타임 실행 → 결과

[미래 GEUL Query]
사용자: "세종대왕 시대 학자 찾아줘"
→ LLM 추론 → SIDX_QUERY(type=person, subtype=scientist, era=medieval, region=east_asia) → NumPy 비트 AND → 결과
```

본질은 동일: **"자연어 의도 → 실행 가능한 구조화 호출"**

SILK 쿼리는 LLM 입장에서 새로운 tool이 추가되는 것과 같다. Tool use 학습과 동일한 방법론 적용 가능.

---

## 10. SILK는 Tool Use를 넘어선다

Tool use의 한계:
```
LLM → 구조화 출력(tool call) → 비구조화 결과(텍스트) → LLM이 다시 해석
```

SILK-native의 차이:
```
LLM → SILK 쿼리 → NumPy 비트 AND → SILK 결과 → LLM이 구조화된 결과 직접 이해
      구조화         구조화         구조화
```

입출력 전체가 SIDX 구조화 형식이면 자연어 변환 없이 의미가 보존된다.
Tool use는 "자연어 속에서 구조화를 잠깐 빌려 쓰는" 것이고, SILK-native는 **사고 자체가 구조화**되는 것이다.

장기 비전 (GEUL 프로젝트):
```
입력 → Encoder → GEUL 심상 → SILK 쿼리 → SILK 결과 → 추론(GEUL) → Decoder → 출력
```

---

## 11. 로드맵

```
Phase 1 (현재)   : 자연어 → JSON → 알고리즘 → mask/pattern → NumPy 비트 AND
                   데이터 축적 단계. 코드북 + JSON 중간층 설계.

Phase 2 (단기)   : SILK를 tool schema로 등록
                   LLM이 tool call처럼 SILK 쿼리 생성.
                   {"type": "person", "region": "east_asia"} → 자동 비트 조립.

Phase 3 (중기)   : SILK 코퍼스로 파인튜닝
                   LLM이 SIDX 의미 구조를 내재화.
                   쿼리뿐 아니라 응답도 구조화.

Phase 4 (장기)   : GEUL-native 사전학습
                   내부 추론 자체가 GEUL 기반.
                   자연어는 I/O 인터페이스일 뿐.
```

---

## 12. 요약

| 질문 | 답 |
|------|-----|
| LLM 전용 모델이 필요한가? | 아니다. 별도 파인튜닝까지는 불필요 |
| 순수 알고리즘으로 가능한가? | 80-90%는 가능. 4개 필드 코드북 사전 룩업 |
| LLM이 아예 불필요한가? | 아니다. 모호성/함축/복합 쿼리에는 범용 LLM 필요 |
| SIDX 비트를 LLM이 직접 생성해야 하나? | 절대 아니다. LLM은 JSON까지만, 비트 조립은 알고리즘 |
| JSON 중간층이 왜 필요한가? | 과도기 검증 + 미래 학습 데이터 축적 |
| 최종 형태는? | LLM이 SILK를 tool처럼 직접 사용, 나아가 GEUL-native 사고 |

---

**문서 종료**
