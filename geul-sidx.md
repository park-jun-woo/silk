# geul-sidx

SIDX 코드북 & 인코딩 파이프라인 — 위키데이터 108.8M 엔티티를 64비트 정수로 인코딩한다.

## GitHub Description

> Codebook builder & encoding pipeline for SIDX (Symbolic Index). Encodes 108.8M Wikidata entities into 64-bit structured identifiers.

## 프로젝트 범위

SILK(검색 엔진)이 소비하는 코드북과 인코딩된 인덱스를 생산한다.

```
geul-sidx (만든다)                silk (쓴다)
├── 타입 정의 (63개)              ├── data/ ← 산출물
├── 비트 할당 (48비트 속성)       ├── silk/ ← 검색
├── 코드북 빌드 (142K 엔트리)     └── index/ ← 인덱스
├── 인코딩 파이프라인
├── 동사 코드북 (WordNet)
└── VALID 검증
```

## 산출물

| 파일 | 내용 |
|------|------|
| `entity_types.json` | 63개 타입 정의 (코드, 이름, 카테고리, QID) |
| `bit_allocation.json` | 타입별 attrs 48비트 필드 배치 (offset, width) |
| `codebook.json` | 타입별 필드→코드 매핑 (142K 엔트리) |
| `verb_codebook.json` | WordNet 동사 코드북 (13,767 synset → 16비트) |
| `fips_to_qid.json` | FIPS 국가코드 → Q-ID 매핑 |
| `cameo_to_wordnet.json` | CAMEO EventCode → WordNet synset 매핑 |

## 현황

- 63개 엔티티 타입 정의 완료
- 5개 타입 속성 인코딩 완료 (Human, Star, Settlement, Organization, Film)
- 108.8M 엔티티 인코딩 완료 (geulwork.entity_sidx)
- 19.7M 미분류 (entity_type=63) — 해결 필요
- 동사 코드북 완성

## 데이터 소스

- **geuldev** DB: 위키데이터 117.4M 엔티티, 17억 트리플
- **geulwork** DB: 인코딩 결과 저장
- **WordNet**: 동사 계층 구조
