# 06. 쿼리 플래너 — 정책 테이블 · 제약 완화 · LLM fallback

> 상태: 설계 초안 (2026-07-06) · 위치: [01-architecture.md](01-architecture.md) ② 컴포넌트의 상세

## 0. 역할과 경계

쿼리 플래너는 **슬롯(깨끗한 구조화 의도) → 실행 가능한 그래프 질의 + DB 필터 + 랭킹 스펙**으로 변환하는 결정적 컴포넌트다.

판단은 세 곳에 나뉘어 있고([05 참조는 아니고 01·본문 §1]), 플래너는 그 사이의 배관이다:

| 판단 종류 | 담당 | 플래너와의 관계 |
|---|---|---|
| 의미 판단 (뭘 원하나) | HyperClova X (의도 파싱) | 플래너의 **입력**(슬롯)을 만든다 |
| 정책 판단 ("안정형"이 뭔가) | 동결된 정책 테이블 | 플래너가 **조회**한다 |
| 트레이드오프 판단 (뭐가 최선인가) | 랭킹 엔진 | 플래너가 **랭킹 스펙**으로 넘긴다 |

**불변 규칙**: 플래너는 LLM을 호출하지 않는다. LLM 개입이 필요한 경우(§4)는 플래너 **앞단**에서 슬롯을 보정한 뒤 다시 플래너로 들어온다. 플래너 자체는 항상 결정적이다.

## 1. 출력 계약 (QueryPlan)

```python
@dataclass
class QueryPlan:
    hard_filters:   list[Filter]        # 관계형 DB WHERE 조건 (통과/탈락)
    graph_targets:  list[GraphTarget]   # 그래프 탐색 시작 노드 (세그먼트/테마/원자재)
    ranking_spec:   RankingSpec          # 랭킹 엔진에 넘길 가중치·정렬 힌트
    relaxation_log: list[RelaxationStep] # 제약 완화가 일어났으면 그 이력 (§3)
    provenance:     PlanProvenance       # 각 필터/타깃이 어느 슬롯·정책에서 나왔는지 (감사용)

@dataclass
class Filter:
    field: str          # 예: 'credit_rating', 'duration', 'total_fee'
    op:    str          # '>=', '<=', 'IN', '=', 'BETWEEN'
    value: object
    origin: str         # 'policy:안정형' | 'constraint:explicit' | 'slot:product_types'
    soft:  bool         # True면 완화 가능 (§3), False면 하드 (절대 못 품)

@dataclass
class GraphTarget:
    node_id:   str      # 세그먼트/테마/원자재 ID (예: 'IT.semi.material')
    direction: str      # '강세' | '약세'
    strength:  float    # 뷰 강도 (없으면 1.0)
```

`provenance`는 03문서의 엣지 provenance와 같은 사상이다 — **왜 이 상품이 추천됐는지**를 사용자 질의 → 슬롯 → 필터/타깃까지 역추적할 수 있어야 한다.

## 2. 정책 테이블 구조

### 2.1 설계 원칙

- **한 번 동결, 재사용**: 정책은 도메인 전문가가 정하고 버전으로 관리한다. 질의마다 재판단하지 않는다.
- **선언적 데이터**: 코드가 아니라 설정(YAML/DB 테이블)으로 둔다. 정책 변경이 배포 없이 가능하고, 변경 이력이 남는다.
- **버전 각인**: 모든 QueryPlan은 사용한 정책 버전을 기록한다 → 사후에 "그때 안정형은 이렇게 정의돼 있었다"를 복원 (point-in-time 원칙).

### 2.2 리스크 정책 테이블 (`RISK_POLICY`)

`risk_level` 슬롯 → 필터 집합. 상품군마다 다른 축을 쓰므로 상품군별로 분기한다.

```yaml
# risk_policy.yaml  (version: v0)
안정형:
  bond:                              # 채권·채권형
    min_credit_rating: "AA-"         # soft
    max_duration_years: 5            # soft
  etf:
    max_volatility_pctile: 50        # soft — 유니버스 내 변동성 하위 50%
    min_aum_krw: 30_000_000_000      # soft — 300억, 유동성 하한
    exclude_leverage_inverse: true   # hard — 레버리지/인버스 원천 제외
  fund:
    max_volatility_pctile: 50        # soft

중립형:
  bond:  { min_credit_rating: "BBB", max_duration_years: 8 }
  etf:   { max_volatility_pctile: 75, exclude_leverage_inverse: true }
  fund:  { max_volatility_pctile: 75 }

공격형:
  bond:  { min_credit_rating: "BB", max_duration_years: null }
  etf:   { exclude_leverage_inverse: false }   # 레버리지 허용
  fund:  { }
```

soft/hard 구분이 §3 완화의 근거가 된다. "안정형인데 레버리지 제외"는 hard(위험의 정의 자체)라 절대 안 풀고, 듀레이션·변동성 상한은 soft라 결과가 없을 때 단계적으로 풀 수 있다.

### 2.3 목적 정책 테이블 (`PURPOSE_POLICY`)

`purpose` 슬롯 → 필터 + 랭킹 힌트. 목적은 "무엇을 걸러낼지"보다 "무엇으로 정렬할지"에 더 영향을 준다.

```yaml
# purpose_policy.yaml
인컴:
  filters:
    - { field: has_distribution, op: "=", value: true, soft: true }
  ranking:
    dividend_yield_weight: high      # 분배율 높을수록 가점
    total_return_weight: low
파킹:                                 # 단기자금 주차
  filters:
    - { field: category, op: "IN", value: [MMF, 초단기채, CMA], soft: false }
    - { field: max_duration_years, op: "<=", value: 0.5, soft: true }
  ranking:
    stability_weight: high
성장:
  filters: []
  ranking:
    total_return_weight: high
    theme_exposure_weight: high      # 그래프 노출도 비중 ↑
절세:
  filters:
    - { field: tax_wrapper, op: "IN", value: [ISA적격, 연금저축, IRP], soft: true }
헤지:
  ranking:
    inverse_correlation_weight: high  # 지정 섹터뷰와 음의 상관 상품 가점
```

### 2.4 뷰 → 그래프 타깃 매핑

`views` 슬롯의 `target`은 **의도 파싱 단계에서 이미 그래프 실재 노드 ID로 확정**돼서 들어온다(§4에서 보장). 플래너는 그대로 `GraphTarget`으로 변환하고, `direction`을 랭킹 부호로 넘긴다.

- 강세: 해당 세그먼트 노출도가 높을수록 가점
- 약세: 노출도가 높을수록 **감점** (또는 헤지 목적이면 음의 노출 가점)

### 2.5 명시적 제약 (`constraints` 슬롯)

사용자가 직접 말한 제약(예: "총보수 0.3% 이하", "달러 환헤지")은 정책보다 **우선**한다. `origin='constraint:explicit'`, 기본 `soft=false`(사용자가 명시한 건 함부로 못 품). 정책과 충돌하면 명시적 제약이 이긴다.

## 3. 제약 완화 규칙

### 3.1 언제 작동하나

플래너가 만든 plan으로 검색한 결과 후보 수가 하한(`MIN_CANDIDATES`, v0: 3) 미만이면 완화 루프에 진입한다. 결과가 충분하면 완화는 일어나지 않는다.

### 3.2 완화 순서 (결정적)

완화는 LLM이 아니라 **고정된 우선순위 규칙**으로 한다. soft 필터에 완화 우선순위(`relax_priority`, 낮을수록 먼저 풂)를 부여하고 순서대로 한 단계씩 푼다.

```
1. 후보 수 < MIN_CANDIDATES 인가? → 아니면 종료
2. 아직 안 푼 soft 필터 중 relax_priority 최소인 것 선택
3. 그 필터를 한 단계 완화 (수치는 정해진 스텝, 범주는 제거)
4. relaxation_log에 (필터, 이전값→새값, 이유) 기록
5. 재검색 → 1로
6. 모든 soft 필터를 풀어도 미달 → "완화 소진" 상태로 종료 (빈 결과 허용)
```

완화 우선순위 기본값(도메인 판단):

| 우선순위 | 대상 | 근거 |
|---|---|---|
| 1 (먼저) | min_aum, 유동성 하한 | 사용자 의도와 가장 무관한 실무적 하한 |
| 2 | 변동성/듀레이션 상한 | 위험 성향의 정도 문제 (한 스텝씩) |
| 3 | 분배 유무 등 목적 필터 | 목적은 완화 시 사용자에게 반드시 고지 |
| 4 (나중) | 신용등급 하한 | 안전의 핵심이라 마지막까지 지킴 |
| 절대 불가 | hard 필터, 명시적 제약 | 레버리지 제외·상품군·사용자 명시 조건 |

수치 완화 스텝은 정책에 정의: 예) 듀레이션 상한 5년 → 7년 → 10년, 신용등급 AA- → A+ → A.

### 3.3 완화의 고지

`relaxation_log`가 비어 있지 않으면 응답 합성 단계(HyperClova X)가 **반드시** 사용자에게 알린다:
> "조건을 모두 만족하는 상품이 적어, 듀레이션 기준을 5년→7년으로 완화한 결과입니다."

완화를 숨기고 결과만 주는 것은 금지 — 사용자가 자기 조건이 지켜졌다고 오해하게 만든다.

### 3.4 상충 조기 감지 (선택, Phase 2+)

완화로도 답이 안 나오는 구조적 상충(예: "AAA 등급인데 배당수익률 8%")은 검색 전에 정책 테이블 간 교차로 감지 가능하다. 감지되면 완화 대신 `clarify`로 되묻는다: "고배당과 최고등급은 양립이 어렵습니다. 어느 쪽을 우선할까요?"

## 4. LLM fallback 경계

### 4.1 원칙: LLM은 플래너 앞에서 "슬롯을 보정"만 한다

플래너는 항상 결정적이다. LLM이 다시 필요한 경우는 **슬롯이 불완전하거나 어휘 밖일 때**이고, 그 처리는 플래너 실행 **전에** 끝나서 깨끗한 슬롯으로 다시 들어온다. LLM은 결코 쿼리·필터·상품명을 직접 생성하지 않는다.

### 4.2 세 가지 fallback 시나리오

| 상황 | 트리거 | LLM 역할 | 출력 제약 |
|---|---|---|---|
| **어휘 밖 뷰 타깃** | `views.target`이 그래프 세그먼트/테마 ID에 매칭 안 됨 | 벡터 검색으로 후보 노드 top-k 제시 → LLM이 그중 **택1 또는 "없음"** | 반드시 실재 노드 ID 중에서만 선택. 새 ID 생성 금지 |
| **슬롯 미충족** | 필수 슬롯(risk_level 등) 결측 | `clarify_needed` 질문 생성 → 사용자에게 되물음 | 자유 서술 가능하나 결과는 슬롯 재파싱으로 회수 |
| **모호·복합 의도** | 하나의 질의에 상충/다중 의도 | 의도를 복수 슬롯 세트로 **분해** 제안 | 각 슬롯 세트는 닫힌 어휘 준수 |

### 4.3 어휘 밖 매칭의 안전장치

가장 위험한 건 첫 번째(어휘 밖 뷰 타깃)다. 근사 매칭이 틀리면 엉뚱한 세그먼트로 검색된다. 안전장치:

1. **닫힌 선택**: LLM에게 벡터 검색이 뽑은 실재 노드 후보 목록을 주고 **그 안에서만** 고르게 한다. 자유 생성 불가.
2. **신뢰도 하한**: 벡터 유사도가 임계(v0: 0.75) 미만이면 자동 선택하지 않고 `clarify`로 사용자 확인.
3. **고지**: 근사 매칭이 적용되면 응답에 명시 — "'○○' 요청을 '△△ 세그먼트'로 해석했습니다."

### 4.4 경계 요약

```
                   ┌─ 슬롯 완전 & 어휘 내 ──────────→ 플래너(결정적) → 검색
질의 → 의도 파싱 →─┤
                   └─ 불완전 / 어휘 밖 ─→ LLM 보정(닫힌 선택·되묻기) ─┐
                                                                    │
                          (보정된 깨끗한 슬롯) ←──────────────────────┘
                                     └────────────────────────────→ 플래너(결정적) → 검색
```

LLM은 루프의 입구 쪽에서만, 항상 선택지를 좁힌 채로 등장한다. 플래너 안으로는 절대 들어오지 않는다.

## 5. 전체 실행 흐름 (의사코드)

```python
def handle_query(user_text, user_profile):
    slots = hyperclova_parse(user_text)              # ① 의도 파싱

    slots = resolve_slots(slots)                      # ④ fallback: 어휘 밖·결측 보정
    if slots.needs_clarification:
        return ask_user(slots.clarify_needed)         # 되묻고 종료

    plan = build_plan(slots, user_profile)            # ② 플래너 (결정적)

    candidates = search(plan)                         # ③ 검색
    while len(candidates) < MIN_CANDIDATES:           # 제약 완화 루프
        step = relax_next(plan)                       # soft 필터 우선순위대로
        if step is None: break                        # 완화 소진
        plan.relaxation_log.append(step)
        candidates = search(plan)

    ranked = rank(candidates, plan.ranking_spec)      # ③ 랭킹 (애널리스트 뷰 가중 포함)

    return hyperclova_synthesize(                     # ① 응답 합성
        ranked, plan.relaxation_log, plan.provenance) # 완화·근거 반드시 고지
```

## 6. Phase별 도입

| Phase | 쿼리 플래너 범위 |
|---|---|
| 1 | 정책 테이블(리스크·목적·상품군) + 뷰→그래프 타깃 + 하드/소프트 필터 + 완화 루프. LLM fallback은 "슬롯 결측 시 되묻기"만 |
| 2 | 어휘 밖 벡터 근사 매칭 fallback, 상충 조기 감지, 랭킹 스펙에 애널리스트 뷰 가중 연결 |
| 3 | 정책 테이블 자체를 성과 신호로 튜닝(어떤 "안정형" 정의가 실제 사용자 만족·성과와 정합했는지) — [05](05-analyst-structure.md) 학습 루프의 확장 |
