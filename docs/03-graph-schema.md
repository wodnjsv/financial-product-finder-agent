# 03. 지식 그래프 스키마

> 상태: 설계 초안 (2026-07-06) · 대상: 프로퍼티 그래프 (Neo4j/Cypher 기준 표기)

## 1. 3계층 구조

그래프는 "누가 쓰는가"와 "얼마나 자주 바뀌는가"로 3계층으로 나눈다. 계층이 곧 신뢰 정책이다.

| 계층 | 내용 | 갱신 주체·주기 | 신뢰 정책 |
|---|---|---|---|
| **Layer 0 — 정적 온톨로지** | GICS 섹터/산업 분류, 밸류체인 세그먼트 DAG, 섹터 간 인과 지도(AFFECTS) | 사람이 하드코딩, 분기~연 단위 개정 | 무조건 신뢰 |
| **Layer 1 — 준정적 마스터** | 상품·기업·지수 노드, 구성종목(HOLDS), 상장/상폐, 기업↔세그먼트 소속 | 배치 파이프라인, 일~월 단위 | 소스 데이터 = 사실로 간주 |
| **Layer 2 — 동적 관계** | 기업 간 공급 관계, 테마 소속, 애널리스트 뷰 | 에이전트 제안 → 검증 게이트 → 커밋 | **출처·신뢰도 필수, 검증 통과분만** |

## 2. 노드 타입

| 노드 | 계층 | 주요 속성 | 비고 |
|---|---|---|---|
| `Sector` | 0 | gics_code, name_kr | 11개 |
| `Industry` | 0 | gics_code, name_kr | GICS 산업(그룹) 수준 |
| `ValueChainSegment` | 0 | segment_id (예: `IT.semi.equipment`), name_kr, sector | [02문서](02-sector-analysts.md)의 세그먼트가 원천 |
| `Commodity` | 0 | name (리튬, WTI, 철광석…), unit | 시계열 저장소의 시리즈 ID 참조 |
| `MacroIndicator` | 0 | source (ECOS/FRED), series_id | 예: ECOS 기준금리, FRED DGS10 |
| `Company` | 1 | ticker, name_kr, listed_market, gics | 상장·주요 비상장 |
| `Index` | 1 | index_id, name, provider | ETF 기초지수 |
| `Product` | 1 | isin, ticker, type (ETF/펀드/채권), 속성은 관계형 DB 참조 | **속성(보수·듀레이션 등)은 그래프에 두지 않는다** |
| `Theme` | 2 | name (AI인프라, 밸류업…) | 에이전트가 생성 가능 — 남발 방지 위해 생성도 검증 게이트 통과 |
| `AnalystView` | 2 | analyst_id, stance, confidence, as_of, valid_until, rationale_ref | 뷰 히스토리를 노드로 보존 (point-in-time 복원용) |
| `Source` | - | type (공시/기사/리포트), url/doc_id, published_at | 벡터 저장소의 원문 청크 참조 |

## 3. 엣지 타입

| 엣지 | 방향 | 계층 | 주요 속성 |
|---|---|---|---|
| `PART_OF` | Industry→Sector, Segment→Sector | 0 | |
| `UPSTREAM_OF` | Segment→Segment | 0 | 밸류체인 DAG 골격 (예: `MA.battery` → `CD.auto`) |
| `AFFECTS` | Macro/Commodity/Segment → Segment/Sector | 0 | sign (+/−), mechanism (한 줄 설명) |
| `EXPOSED_TO` | Segment→Commodity | 0 | 예: `MA.battery` → 리튬 |
| `BELONGS_TO` | Company→Industry | 1 | |
| `OPERATES_IN` | Company→Segment | 1→2 | weight (매출 비중 근사). 초기엔 업종분류로 근사(L1), 이후 에이전트가 세분화(L2) |
| `TRACKS` | Product→Index | 1 | |
| `HOLDS` | Product→Company | 1 | weight, as_of — **스냅샷 시리즈로 보존** (리밸런싱 이력) |
| `SUPPLIES` | Company→Company | 2 | product, share 추정, **grounding 필수** |
| `IN_THEME` | Company/Product→Theme | 2 | grounding 필수 |
| `HAS_VIEW` | AnalystView→Segment/Sector/Theme | 2 | |
| `GROUNDED_BY` | (모든 L2 엣지·노드)→Source | - | 검증의 물리적 형태 |

## 4. 엣지 메타데이터 규약 (전 엣지 공통)

```
created_by:   ingest-pipeline | analyst:{id} | human
created_at:   timestamp
valid_from:   date        # 관계가 유효해진 시점
valid_to:     date|null   # null = 현재 유효. 만료 시 삭제하지 않고 valid_to를 채운다
confidence:   0.0~1.0     # Layer 2만. Layer 0/1은 1.0 고정
```

**시간성 원칙**: 엣지는 절대 덮어쓰거나 삭제하지 않는다. 변경 = 기존 엣지 `valid_to` 마감 + 새 엣지 추가.
→ "그 시점의 밸류체인/구성종목"을 항상 복원할 수 있고, 이것이 사후 성과 평가(Phase 3)의 전제가 된다.

## 5. Layer 2 검증 게이트

에이전트가 제안한 엣지가 커밋되는 조건:

1. **출처 존재**: `GROUNDED_BY` 엣지로 연결될 Source(공시·기사·리포트 원문 청크)가 실재해야 한다.
2. **출처-주장 정합 검증**: 제안 에이전트와 별개의 검증 단계(독립 LLM 호출)가 "이 출처가 이 관계를 실제로 지지하는가"를 판정.
3. **신뢰도 하한**: confidence < 0.5는 커밋하지 않고 보류 큐에 적재 (추가 출처 확보 시 재평가).
4. **충돌 감지**: 기존 유효 엣지와 모순되면(예: 동일 관계의 반대 방향) 자동 커밋하지 않고 사람 리뷰 큐로.

## 6. 대표 질의 패턴

**정방향 — "이 ETF는 뭐에 노출돼 있어?"**
```cypher
MATCH (p:Product {ticker:$t})-[h:HOLDS]->(c:Company)-[o:OPERATES_IN]->(s:ValueChainSegment)
WHERE h.valid_to IS NULL AND o.valid_to IS NULL
RETURN s.segment_id, sum(h.weight * o.weight) AS exposure
ORDER BY exposure DESC
```

**역방향 — "리튬 리스크를 피하려면 어떤 상품을 피해야 해?"**
```cypher
MATCH (cm:Commodity {name:'리튬'})<-[:EXPOSED_TO]-(s:ValueChainSegment)
      <-[o:OPERATES_IN]-(c:Company)<-[h:HOLDS]-(p:Product)
WHERE h.valid_to IS NULL AND o.valid_to IS NULL
RETURN p.ticker, sum(h.weight * o.weight) AS lithium_exposure
ORDER BY lithium_exposure DESC
```

**매크로 — "금리 인하 수혜 상품"**: `MacroIndicator(기준금리)` → `AFFECTS(sign=−)` → 수혜 Sector/Segment 집합 → 노출 상품. (금리↓ 수혜 = sign이 −인 대상)

**랭킹 엔진과의 계약**: 그래프는 `(product, segment, exposure, 경로)` 튜플을 반환하고, 랭킹 엔진이 여기에 애널리스트 뷰(`HAS_VIEW`)와 하드필터 결과를 결합한다. 경로 자체가 추천 근거 설명(explanation trace)의 재료가 된다.

## 7. 저장소 간 참조 규약

- 그래프 노드는 **관계(위상)만** 갖는다. 수치·텍스트의 원본은:
  - 상품 속성 → 관계형 DB (`Product.isin`으로 조인)
  - 가격·금리 시계열 → 시계열 저장소 (`MacroIndicator.series_id`, `Commodity.series_id`)
  - 원문 → 벡터 저장소 (`Source.doc_id`)
- GraphRAG 조합: 질의 응답 시 그래프가 경로(구조 맥락)를, 벡터 저장소가 원문 근거(비정형 맥락)를 공급한다.
