# SILK 코드북 현황

SILK 아키텍처가 사용하는 코드북 작업 상태.

---

## 동사 코드북 — 완성

WordNet 13,767개 동사 synset → 16비트 의미정렬 코드북.

| 항목 | 수치 |
|------|------|
| 총 동사 | 13,767 |
| 고유 16비트 코드 | 13,767 (충돌 0) |
| 분류 체계 | 10 Primitive → 68 Sub-primitive → 559 Root → 13,767 Leaf |
| 인코딩 | Huffman 유사 가변길이 (4~8비트 prefix + DFS index) |
| 최종 산출물 | `verb_bits.json` |

### 분류 체계

```
BE(99)        — 상태 유지 (exist, have, remain)
PERCEIVE(38)  — 지각 (see, hear, feel)
FEEL(56)      — 감정 (love, hate, fear)
THINK(51)     — 사고 (think, know, believe)
CHANGE(96)    — 상태 변화 (become, die, begin)
CAUSE(277)    — 사역/행위 (make, do, put)
MOVE(49)      — 이동 (go, come, travel)
COMMUNICATE(48) — 소통 (say, tell, speak)
TRANSFER(33)  — 전달/이전 (give, receive, send)
SOCIAL(52)    — 사회적 (meet, join, follow)
```

### Graceful Degradation

```
bit[1-4]:   Primitive 수준 (10종)
bit[1-8]:   Sub-primitive 수준 (68종)
bit[1-16]:  개별 동사 (13,767종)
```

---

## 엔티티 코드북 — 진행 중

Wikidata 108.8M 개체 → 18비트 분류 (type 6 + subtype 5 + region 4 + era 3) + Q-ID 32비트.

| 항목 | 수치 |
|------|------|
| SIDX 대상 | 108,854,572 (Wikimedia 내부 제외) |
| type | 64종 (6비트) |
| subtype | 타입별 32종 (5비트) |
| region | 16권역 (4비트) |
| era | 8시대 (3비트) |
| SIDX 생성 | 108,878,520 (100%) |
| 속성 인코딩 | 17,442,999 (16.0%) |

### 타입별 진행

| 타입 | 이름 | 개체 수 | 속성 인코딩 | 비율 |
|------|------|---------|-------------|------|
| 0x00 | Human | 12,553,670 | 12,182,686 | 97.0% |
| 0x01 | Taxon | 3,904,250 | 3,531,305 | 90.5% |
| 0x0C | Star | 4,843,949 | 1,729,008 | 35.7% |
| 0x31 | Document | 45,000,000 | 24,478 | 3.5% |
| 0x3F | Unknown | 19,678,178 | 0 | 0.0% |

### 잔여 과제

**긴급 — 62.2% 분류 실패 해소:**
- Q6256(Country), Q3624078(Sovereign State) → primary_mapping.json 추가
- Q5864(G-type Star) → Star 매핑 추가
- Q55983715(Common Name Taxon) → Taxon 매핑 추가
- P279 체인 탐색 구현 (최대 5홉)

**중기 — 스키마 정제:**
- 과도한 세분화 타입 통합 (Settlement/Village/Hamlet 등)
- 누락 타입 추가 (Country, City, University 등)
- Property 매핑률 43.9% → 70%+ 개선

### 검색 아키텍처 (설계 완료)

```
사전 구축: ~35,000 leaf entries (~1MB, L2 캐시 적재)
  ("Human", "country", "Korea") → { mask: 0x..., value: 0x... }

쿼리 파이프라인:
  사용자 쿼리 → 소형 LLM (의미 파싱) → Dictionary (마스크 조립) → SIMD 스캔
```

### 참조 데이터

| 파일 | 크기 | 용도 |
|------|------|------|
| `entity_types_64.json` | 11.7KB | 64개 타입 정의 + QID 매핑 |
| `type_schemas.json` | 83KB | 48비트 속성 스키마 |
| `codebooks_full.json` | 283KB | QID → 코드 매핑 |
| `type_mapping.json` | 7.7KB | 하위타입 → 64타입 매핑 |

---

## SILK PoC에서의 활용

```
SILK 인코딩:
  - 동사: verb_bits.json (16비트, 즉시 사용 가능)
  - 엔티티: 18비트 분류 (type 6 + subtype 5 + region 4 + era 3) + Q-ID 32비트
  - 코드북: type/subtype/region/era 각 필드 유효값 정의 (sidx.yaml)

SILK 검색:
  - 64비트 SIDX: [type 6 | subtype 5 | region 4 | era 3 | 미사용 14 | Q-ID 32]
  - Multi-SIDX: 문서당 복수 SIDX → NumPy 비트 AND → 교집합 → 원문 로드
  - 검색 = index[(index & mask) == pattern]
```
