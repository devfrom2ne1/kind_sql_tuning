## 4. 조인 튜닝

## 4.1 NL 조인

### 4.1.1 기본 메커니즘

#### Outer Table과 Inner Table

```sql
select e.사원명, c.고객명, c.전화번호
from 사원    e,    -- Outer Table(= Driving Table)
     고객번호 c.    -- Inner Table
```

- 기본적으로 NL조인은 Outer와 Inner 양쪽 테이블 모두 인덱스를 이용한다.
	- Outer Table은 사이즈가 크지 않으면 인덱스를 이용하지 않을 수 있다.

- Outer Table : 1번 Sacn
- Inner Table : Outer Table에서 읽은 건수만큼 반복 Scan

### 4.1.2 NL 조인 실행계획 제어

#### NL조인 실행계획 읽는 법

```
Exdrecution Plan
-------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS
1	0   NESTED LOOPS
2	1	 TABLE ACCESS (BY INDEX ROWID) OF '사원' (TABLE)
3	2       INDEX (RANGE SCAN) OF '사원_X1' (INDEX)
4	3     TABLE ACCESS (BY INDEX ROWID) OF '고객' (TABLE)
5	4       INDEX (RANGE SCAN) OF '고객_X1' (INDEX)
```

- 위쪽테이블 (사원) = Outer Table = Driving Table
- 아래테이블 (고객) = Inner Table

#### NL조인 힌트 기술하는 방법

1. **use_nl 힌트**

```sql
select /*+ use_nl(A, B, C, D) */ *
from A, B, C, D
where ...
```

- `use_nl` : 네 개 테이블을 NL방식으로 조인해라 
- `(A, B, C, D)` : 순서는 옵티마이저가 스스로 정하도록 맡긴 것이다.

2. **ordered 힌트**

```sql
-- (1) e -> c 순으로 NL조인 
-- 사원 e = Outer(Driving) / 고객 c = Inner
select /*+ ordered use_nl(c) */
  e.사원명, c.고객명, c.전화번호
from 사원 e, 고객 c
where e.입사일자 >= '19960101'
and c.관리사원번호 = e.사원번호

-- (2) A -> B ->  C -> D 순으로 조인하라
--      (NL) (NL) (Hash)
select /*+ ordered use_nl(B) use_nl(C) use_hash(D)
from A, B, C, D
```

- `ordered` : FROM절에 기술한 순서대로 조인하라고 옵티마이저에게 지시할 때 사용하는 힌트

3. **leading 힌트**

```sql
select /*+ leading(A, B, C, D) use_nl(A) use_nl(D) use_hash(B) */ *
from A, B, C, D
where ...
```

- `leading` : FROM절 순서에 관계 없이, leading 뒤에 기술한 순서대로 조인하라고 옵티마이저에게 지시하는 힌트


### 4.1.3 NL 조인 수행 과정 분석

```sql
select /*+ ordered use_nl(c) index(e) index(c) */
	...
from 사원 e, 고객 c
where c.관리사원번호 = e.사원번호 --- (1)
and   e.입사일자 >= '19960101' --- (2)
and   e.부서코드 = 'Z123'      --- (3)
and   c.최종주문금액 >= 20000   --- (4)
```

```
* 사원_PK : 사원번호
* 사원_X1 : 입사일자
* 고객_PK : 고객번호
* 고객_X1 : 관리사원번호
* 고객_X2 : 최종주문금액
```

- 조건절 비교 순서 : (2) → (3) → (1) → (4)
- 사용한 인덱스
	- (2) : `사원_X1`(입사일자)
	- (1) : `고객_X1`(관리사원번호)

- 왜 `고객_X2`(최종주문금액)는 안 탈까?
	- NL 조인에서 후행 테이블은 조인 연결고리 컬럼(관리사원번호) 인덱스를 타는 것이 정석입니다. 
	- `고객_X2`를 타려면 조인이 아니라 고객 테이블을 먼저 읽어야 하는데, 힌트에서 사원을 먼저 읽으라고 지정했기 때문입니다.

### 4.1.4 NL 조인 튜닝 포인트

1. Outer Table 인덱스을 읽고 나서 Outer Table을 액세스 하는 부분을 최소화해야 한다. 
2. Inner Table 인덱스를 통해 액세스 탐색하는 횟수(= 조인 액세스 획수)가 적게 해야 한다. 
3. Inner Table 인덱스를 읽고 나서 Inner Tabel을 액세스 하는 비율을 최소화해야 한다.

### 4.1.5 NL 조인 특징 요약

#### 1. NL조인은 랜덤 액세스 위주의 조인 방식이다.
- '[랜덤액세스](https://github.com/devfrom2ne1/kind_sql_tuning/blob/main/etc/random-access.md)'은 레코드 하나를 읽으려고 블록을 통째로 읽는 방식이다.
- 그래서 인덱스 구성이 아무리 완벽해도 대량 데이터 조인할 때 NL 조인이 불리하다.

#### 2. NL조인은 한 레코드씩 순차적으로 진행한다. 
- 부분범위  처리가 가능하다면 빠른 응답 속도를 낼 수 있다.

#### 3. NL조인은 인덱스 구성 전략이 중요하다.
- 조인 컬럼에 대한 인덱스 유무, 구성 방식 등에 따라 조인 효율이 달라진다.

#### 4. NL조인은 온라인 트랜젝션 처리(OLTP)에 적합하다.
- 소량데이터나, 부분범위처리가 가능할 때 유리하기 때문이다.

### 4.1.6 NL 조인 튜닝 실습

### 4.1.7 NL 조인 확장 매커니즘

#### 테이블 Prefetch

> [!NOTE]
> Prefetch는 **"Inner 테이블을 먼저 읽는 것"** 이 아니라, **"Inner 테이블의 인덱스를 읽는 시점에, 곧 필요해질 테이블 블록들을 미리 예측해서 메모리에 로드하는 기술"** 입니다.
> 덕분에 디스크 I/O를 기다리는 시간(db file sequential read)이 획기적으로 줄어들게 됩니다.

| 단계 | 전통적인 NL 조인 | Prefetch 적용 NL 조인 |
|---|---|---|
| 1단계 | Outer 한 건 읽기 | Outer 한 건 읽기 |
| 2단계 | Inner 인덱스 한 건 찾기 | Inner 인덱스 여러 건(ROWID) 미리 확인 |
| 3단계 | 해당 테이블 블록 1개 읽기 | 미리 확인한 ROWID들의 블록들을 한꺼번에 캐싱 |
| 4단계 | 조인 수행 | 캐시된 블록에서 즉시 조인 수행 |

- Prefetch는 조인 순서(Outer → Inner) 자체가 바뀌는 것은 아닙니다. 	
	- 드라이빙 테이블(Outer)을 먼저 읽어야 드리븐 테이블(Inner)을 찾을 수 있다는 NL 조인의 기본 원칙은 그대로 유지됩니다.
- 헷갈리실 수 있는 부분은 "데이터를 디스크에서 퍼 올리는 시점" 때문일 거예요. 
- Prefetch가 '먼저' 하는 것
	- 전통적인 NL 조인은 **[인덱스 한 건 읽기 → 테이블 한 건 읽기]** 를 무한 반복합니다. 
	- 하지만 Prefetch는 이 순서를 살짝 비틉니다.

- Prefetch 순서
	- Outer 테이블에서 조인할 조건의 행을 하나 읽습니다.
 	- Inner 테이블의 인덱스를 탐색하여 테이블의 주소(ROWID)를 찾습니다.
	- **여기서 핵심** : 찾은 ROWID를 가지고 바로 테이블로 달려가는 게 아니라, 인덱스 리프 블록에 있는 다음 ROWID들을 미리 훑어봅니다.
	- "어차피 다음 루프에서 이 블록들이 필요하겠네?"라고 판단되면?
	- 실제 조인이 일어나기 '전'에 해당 테이블 블록들을 디스크에서 버퍼 캐시로 한꺼번에(Parallel/Batch) 퍼 올립니다.
> 결론: 조인 순서가 바뀌는 게 아니라, **"Inner 테이블의 데이터 블록을 실제 조인 단계가 오기 전에 미리 메모리에 갖다 놓는 것"** 입니다.

- 왜 실행계획에서는 Inner Table이 위로 가 보일까?
	- 이 구조 때문에 실행계획의 모양이 바뀌어서 오해하기 쉽습니다.
	- 일반 NL 조인 : NESTED LOOPS가 부모고, 그 아래에 Outer와 Inner(Index+Table)가 자식으로 붙음.
 	- Prefetch 적용: TABLE ACCESS(Inner)가 NESTED LOOPS보다 위(부모)에 위치함.
	- 이것은 **"조인이 완료된 결과(ROWID 세트)를 부모 노드인 TABLE ACCESS 연산에 던져주면, 부모가 블록을 한꺼번에 퍼 올린다"** 는 처리 흐름을 보여주는 것이지, 
	- 테이블을 먼저 읽는다는 뜻이 아닙니다.

- 오라클 11g 이상부터는 `NLJ_PREFETCH`보다 더 강력한 `NLJ_BATCHING`이 기본적으로 작동하는 경우가 많습니다. 
- 정렬(Order) 문제
	- `no_nlj_prefetch`를 쓰는 가장 큰 이유 중 하나는 데이터가 인덱스 정렬 순서 그대로 나오길 기대할 때입니다. 
	- Prefetch나 Batching이 들어가면 미세하게 결과 순서가 바뀔 수 있기 때문입니다.

#### nlj_prefetch 힌트 사용 예시

```sql
SELECT /*+ LEADING(o) USE_NL(i) NLJ_PREFETCH(i) */
       o.order_date, i.product_id, i.quantity
FROM   orders o, order_items i
WHERE  o.order_id = i.order_id
  AND  o.customer_id = :cust_id;
```

```
--------------------------------------------------------------------------------------
| Id  | Operation                    | Name          | Rows  | Bytes | Cost (%CPU)|
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |               |    10 |   500 |    25   (0)|
|   1 |  TABLE ACCESS BY INDEX ROWID | ORDER_ITEMS   |     2 |    40 |     2   (0)|
|   2 |   NESTED LOOPS               |               |    10 |   500 |    25   (0)|
|   3 |    TABLE ACCESS FULL         | ORDERS        |     5 |   150 |    15   (0)|
|*  4 |    INDEX RANGE SCAN          | ITEM_ORDER_IX |     2 |       |     1   (0)|
--------------------------------------------------------------------------------------
```

- `nlj_prefetch` 힌트는 오라클이 판단하기에 Prefetch 효율이 낮다고 생각하여 일반적인 NL 조인을 하려고 할 때, **"아니야, 테이블 블록을 미리 좀 퍼 올려줘"** 라고 강제할 때 씁니다.
- `nlj_prefetch`가 성공적으로 적용되면, TABLE ACCESS가 NESTED LOOPS보다 위로 올라가는 형태가 됩니다.
- `TABLE ACCESS BY INDEX ROWID`가 Id 2번(NESTED LOOPS)의 결과물을 받아서 처리하는 부모 노드 역할을 합니다.

#### no_nlj_prefetch 힌트 사용 예시

```sql
SELECT /*+ LEADING(o) USE_NL(i) NO_NLJ_PREFETCH(i) */
       o.order_date, i.product_id, i.quantity
FROM   orders o, order_items i
WHERE  o.order_id = i.order_id
  AND  o.customer_id = :cust_id;
```

```
--------------------------------------------------------------------------------------
| Id  | Operation                    | Name          | Rows  | Bytes | Cost (%CPU)|
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |               |    10 |   500 |    25   (0)|
|   1 |  NESTED LOOPS                |               |    10 |   500 |    25   (0)|
|   2 |   TABLE ACCESS FULL          | ORDERS        |     5 |   150 |    15   (0)|
|   3 |   TABLE ACCESS BY INDEX ROWID| ORDER_ITEMS   |     2 |    40 |     2   (0)|
|*  4 |    INDEX RANGE SCAN          | ITEM_ORDER_IX |     2 |       |     1   (0)|
--------------------------------------------------------------------------------------
```

- 반대로, Prefetch 기능 때문에 오히려 성능이 떨어지거나, 실행 계획을 아주 전통적인(Classical) NL 조인 형태로 고정하고 싶을 때 사용합니다.
- Prefetch를 끄면 우리가 흔히 아는 가장 기본적인 NL 조인 구조로 돌아옵니다.
- NESTED LOOPS가 가장 위에 있고, 그 아래에 바로 TABLE ACCESS가 자식 노드로 붙어 있습니다. 
- 한 건 읽을 때마다 바로 테이블로 가는 순차적인 방식입니다.

#### 배치 I/O

* 참고 : [부분범위처리와 배치 I/O](https://github.com/devfrom2ne1/kind_sql_tuning/blob/main/chapter3/chapter3.2.md)

- 배치 I/O는 꼭 힌트를 써야만 작동하는 것은 아닙니다. 
- 현대의 오라클(11g 이후 버전)은 **비용 기반 옵티마이저(CBO)** 가 판단했을 때 배치 I/O를 사용하는 것이 더 빠르다고 생각하면 자동으로 이 방식을 선택합니다.
- 하지만 실무에서는 옵티마이저가 내 의도와 다르게 작동할 수 있기 때문에, 힌트를 통해 제어해야 하는 상황이 분명히 있습니다. 

#### 1. 자동으로 작동하는 경우 (Default)
- 오라클 11g 이상에서는 특별한 설정을 하지 않아도 다음 조건이 충족되면 배치 I/O가 활성화됩니다.
 	* 옵티마이저 모드가 ALL_ROWS인 경우.
 	* 통계 정보상 드라이빙 테이블에서 읽을 행이 많아, 후행 테이블의 인덱스 랜덤 액세스 부하가 클 것으로 예상될 때.
 	* 내부 파라미터(_nlj_batching_enabled)가 기본값(1)으로 설정되어 있을 때.

#### 2. 힌트를 써야 하는 경우 (Control)

```sql
SELECT /*+ LEADING(a) USE_NL(b) NLJ_BATCHING(b) */ * 
FROM table_a a, table_b b ...
```

- 옵티마이저가 일반 NL 조인을 선택했는데, 개발자가 판단하기에 "이건 랜덤 액세스가 너무 많으니 배치로 몰아서 읽는 게 유리하겠다" 싶을 때 강제로 사용합니다.
 	* 배치 I/O 강제: /*+ NLJ_BATCHING(테이블명) */
 	* 배치 I/O 방지: /*+ NO_NLJ_BATCHING(테이블명) */ (데이터 정렬 순서가 중요할 때 주로 사용)

> [!WARNING]
> 배치 I/O 힌트 단독으로 쓰기보다는 조인 순서(LEADING)와 조인 방식(USE_NL) 힌트를 함께 쓰는 것이 정확합니다.

#### 3. 배치 I/O가 작동하지 않는 상황들
힌트를 써도 다음과 같은 경우에는 작동하지 않을 수 있습니다.

| 상황 | 이유 |
|---|---|
| 인덱스만으로 조회 가능 | 테이블에 접근할 필요가 없으면(Index Only Scan) I/O 배칭 자체가 필요 없음 |
| 데이터 건수가 너무 적음 | 한두 블록만 읽으면 되는데 배치로 모으는 오버헤드가 더 크다고 판단될 때 |
| 결과 정렬이 우선순위임 | ORDER BY가 인덱스 순서에 의존하고 있을 때, 배칭을 하면 순서가 뒤섞여 옵티마이저가 기피할 수 있음 |
| 구버전 오라클 | 10g 이하 버전에서는 이 메커니즘 자체가 존재하지 않음 (대신 Prefetch만 존재) |

#### 4. 내 쿼리가 배치 I/O를 썼는지 확인하는 법
- 실행 계획(Execution Plan)에서 다음 키워드를 찾으시면 됩니다.
	- NESTED LOOPS가 두 번 나타남: 상위 NL과 하위 NL로 분리되어 있다면 배칭이 일어난 것입니다.
 	* BATCHED 키워드: `TABLE ACCESS BY INDEX ROWID BATCHED`라고 명시되어 있다면 확실히 배치 I/O가 작동한 것입니다.

