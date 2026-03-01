# SILK

**Symbolic Index for LLM Knowledge** — A neuro-symbolic search architecture.

Search with 64-bit integers. No vector DB, no ANN graph, no embedding model.

Part of the [GEUL](https://geul.org) project.

> [Korean version (한국어)](README.ko.md)

## Core Thesis

Search was never a hard problem — it was a **symptom of unstructured data**.
Assign 64-bit structure at write time, and the symptom disappears.

```python
results = index[(index & mask) == pattern]
```

Sub-second search over 100M Wikidata entities in 1.3GB of memory.
Pure Python (NumPy) outperforms optimized C++/Rust vector DBs — an architectural win.

## Why Not Vector Embeddings

Vector embeddings destroy structure.

```
"Yi Sun-sin was a military commander of the Joseon Dynasty"
  → A human instantly grasps: Korean, military, 16th century

Embedding model:
  [0.234, -0.891, 0.445, ..., 0.112]  (384-dim float)
  → Structure gone. Can't tell if it's a person or a location.

Then, to recover the destroyed structure:
  ANN graphs (HNSW, IVF-PQ), cross-encoders, rerankers, metadata filters...
```

SILK preserves structure.

```
SIDX: [Human / military / east_asia / early_modern]
  → Structure lives in the bits. Human-readable.
  → Nothing to recover. It was never destroyed.
```

The key insight is an inversion of order.

```
Conventional:  write first → structure later (indexing)
SILK:          structure at write time → search is free
```

## SIDX Bit Layout

SIDX follows the [GEUL standard](https://geul.org/grammar/) Entity Node specification.

```
[prefix 7 | mode 3 | entity_type 6 | attrs 48]
 MSB(63)                                  LSB(0)
```

| Field | Bits | Width | Description |
|-------|------|-------|-------------|
| prefix | 63-57 | 7 | GEUL protocol header (`0001001` fixed, ignored during search) |
| mode | 56-54 | 3 | Quantification/number mode (registered=0, definite, universal, existential, etc.) |
| entity_type | 53-48 | 6 | 64 top-level types (Human=0 ... Election=62, unclassified=63) |
| attrs | 47-0 | 48 | Type-specific attribute encoding (codebook-defined) |

Search target: entity_type 6 bits + attrs 48 bits = **54 bits**.
QIDs are not embedded in SIDX; they are stored in a separate parallel array.

### attrs 48 bits — Per-Type Schema

```
Human(0):        subclass 5 | occupation 6 | country 8 | era 4 | decade 4 | gender 2 | notability 3 | ...
Star(12):        constellation 7 | spectral_type 4 | luminosity 3 | magnitude 4 | ra_zone 4 | dec_zone 4 | ...
Settlement(28):  country 8 | admin_level 4 | admin_code 8 | lat_zone 4 | lon_zone 4 | population 4 | ...
Organization(44): country 8 | org_type 4 | legal_form 6 | industry 8 | era 4 | size 4 | ...
Film(51):        country 8 | year 7 | genre 6 | language 8 | color 2 | duration 4 | ...
```

5 types have full attribute bit allocations. The remaining types encode entity_type only.

## Architecture

SILK has two pipelines: **encoding** (at write time) and **search** (at query time).
Both follow the same principle: symbolic components handle structure, LLMs handle meaning.

```
Encoding Pipeline (write time)
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  LLM Tagging  │───►│  VALID Check  │───►│   Codebook    │
│  doc → JSON   │    │  codebook     │    │   Encoding    │
│  (semantic    │    │  validation   │    │  JSON → SIDX  │
│   classify)   │    │  reject bad   │    │  (bit pack)   │
└──────────────┘    └──────────────┘    └──────────────┘
  LLM tags           code validates     codebook-based encode

Search Pipeline (query time)
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Query Parse   │───►│ Bit AND Filter│───►│  LLM Judge   │
│ codebook      │    │ NumPy SIMD   │    │  few cands    │
│ lookup + LLM  │    │ 100M → tens  │    │  tens → answer│
└──────────────┘    └──────────────┘    └──────────────┘
  extract meaning    filter wide         narrow to answer
```

Core strategy: **SIDX bit AND filters wide without missing anything**, then **LLM judges only the few remaining candidates**.

```
Each component does what it's best at:
  Human: design structure (codebook)
  LLM:   semantic classification (tagging) + semantic judgment (search)
  Code:  rule validation (VALID) + bit packing
  CPU:   bulk comparison (NumPy SIMD)
```

### Encoding: LLM Tagging → VALID Verification

An LLM reads documents and produces JSON tags. VALID performs mechanical validation against the codebook.
Values not in the codebook, cross-type consistency violations, and constraint violations **physically cannot enter the index**.

```
LLM tags:  {"type": "Human", "occupation": "military", "country": "korea"}
VALID:     occupation="military" ∈ codebook? ✓  country="korea" ∈ codebook? ✓
           Human with constellation field? ✗ rejected
Encode:    codebook maps "military"→6 bits, "korea"→8 bits → bit pack → SIDX uint64
```

LLMs can hallucinate. But VALID acts as a gatekeeper, so index contamination is impossible.
Only JSON that passes VALID is encoded into a SIDX 64-bit integer via the codebook.

### Search: Filter Wide, LLM Narrows

```
1. Query semantics     codebook lookup 80% / LLM-assisted 20%
2. Bitmask assembly    deterministic — algorithmic
3. NumPy bit AND       deterministic — full scan 100M in 20ms
4. LLM final judgment  few candidates only — semantic precision
```

### 80% of Searches Are Structural Queries

```
"Yi Sun-sin"             → Q484523 exact match. Bit AND. Done.
"Samsung news"           → org/company + doc_meta/news. Bit AND. Done.
"Biden-Xi summit"        → Q6279 ∩ Q15031 ∩ meeting. Intersection. Done.
```

```
Structural queries (80%):  SILK bit AND alone. No LLM needed.
Semantic queries (15%):    SILK narrows candidates → LLM judges 5-10.
Generative queries (5%):   SILK locates documents → LLM generates.
```

## Multi-SIDX

A single document/event carries multiple SIDXs.

```
News article "Samsung, NVIDIA, Hyundai Motor chiefs meet":

SIDX[0]: [Human / business  / east_asia ]  Jay Y. Lee
SIDX[1]: [Org   / company   / east_asia ]  Samsung
SIDX[2]: [Human / business  / n_america ]  Jensen Huang
SIDX[3]: [Org   / company   / n_america ]  NVIDIA
SIDX[4]: [Human / business  / east_asia ]  Euisun Chung
SIDX[5]: [Org   / company   / east_asia ]  Hyundai Motor
SIDX[6]: [Event / meeting   / east_asia ]  summit
```

All the same 64-bit SIDX. Same index. Same bit AND search.
The entity_type field distinguishes entities (Human, Org) from events (Event).

## Index Structure

```python
sidx_array = np.array([...], dtype=np.uint64)   # 108.8M × 8B = 870MB
qid_array  = np.array([...], dtype=np.uint32)   # 108.8M × 4B = 435MB
# Total ~1.3GB in memory
```

```
Index construction:
  Elasticsearch: tokenize → analyze → inverted index → segment merge
  Pinecone:      embed → HNSW graph → clustering
  SILK:          sort
```

No data structures. One sorted array.

## Comparison with Vector DBs

| | SILK | Vector DB |
|------|------|-----------|
| Index size (1T entries) | 12TB | 1.5PB (125x) |
| Index build | sort | HNSW, days |
| Cold start | open file, ready | graph build, hours |
| Partitioned scan | yes (same results) | no (graph breaks) |
| Compound conditions | intersection (exact) | compressed into 1 vector (approximate) |
| Results | exact (set operations) | approximate (similarity ranking) |
| Auditability | JSON (white-box) | impossible (black-box) |

```
Bit AND is order-independent, stateless, and additively composable.
Vector DB: entire graph must be in memory. Partition = destruction.
SILK:      split anywhere. Partition = slower, but same results.
```

## Quality Assurance Pipeline

Vector embeddings are black-box and cannot be audited. SIDX is JSON and fully auditable.

```
Stage 1 — Small LLM tagging     Llama 8B / GPT-4o-mini. Accuracy 85-90%.
Stage 2 — VALID machine check    Valid values, consistency, constraints. Cost $0.
Stage 3 — Large LLM review       confidence=low only. Accuracy 99%+.

VALID is the gatekeeper: hallucinations outside codebook values physically cannot enter the index.
```

## Installation

```bash
pip install -e .
```

### Dependencies

- Python >= 3.10
- NumPy

## Project Structure

```
silk/
├── silk/           # Core library
│   ├── codebook.py # Codebook load, lookup, bit packing
│   ├── encoder.py  # tag → SIDX uint64
│   ├── valid.py    # VALID verification
│   ├── search.py   # NumPy bit AND
│   └── query.py    # query → mask/pattern
├── data/           # Codebooks, mapping tables (GEUL outputs)
├── scripts/        # Data pipelines
├── index/          # Generated index (gitignored)
├── benchmarks/     # FAISS comparison benchmarks
├── notebooks/      # Jupyter demos
└── paper/          # Paper
```

## License

MIT
