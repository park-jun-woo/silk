# SILK

**Symbolic Index for LLM Knowledge** — 뉴로-심볼릭 검색 아키텍처.

64비트 정수로 검색한다. 벡터 DB, ANN 그래프, 임베딩 모델이 필요 없다.

## 핵심 아이디어

모든 엔티티를 64비트 정수(SIDX)로 인코딩한다.
검색은 NumPy 비트 AND 한 줄이다.

```python
results = index[(index & mask) == pattern]
```

위키데이터 1억 개 엔티티를 1.3GB 메모리에서 서브초 검색.
Python(NumPy)만으로 최적화된 C++/Rust 벡터 DB를 이긴다 — 아키텍처의 승리.

## SIDX 비트 레이아웃

```
[prefix 7 | mode 3 | entity_type 6 | attrs 48]
 MSB(63)                                  LSB(0)
```

- `entity_type`: 63개 타입 (Human, Star, Settlement, Organization, Film, ...)
- `attrs`: 타입별 속성 인코딩 (국가, 직업, 장르 등)
- 검색 대상: entity_type 6비트 + attrs 48비트 = **54비트**

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
├── data/           # 코드북, 매핑 테이블
├── scripts/        # 데이터 파이프라인
├── index/          # 생성된 인덱스 (gitignored)
├── benchmarks/     # FAISS 비교 벤치마크
├── notebooks/      # Jupyter 데모
└── paper/          # 논문
```

## 아키텍처

```
심볼릭 (규칙)              뉴럴 (LLM)
├── 코드북                  ├── 소형 LLM 태깅
├── VALID 검증              ├── 대형 LLM 검수
├── NumPy 비트 AND          └── LLM 최종 판정
└── 정렬 + 이진 탐색
```

검색 파이프라인:

1. 쿼리 의미 추출 (코드북 룩업 80% / LLM 보조 20%)
2. 비트마스크 조립 (결정적)
3. NumPy 비트 AND (결정적)
4. LLM 최종 판정 (소수 후보만)

## 라이선스

MIT
