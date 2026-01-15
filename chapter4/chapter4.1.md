# 4. 조인 튜닝

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
0		SELECT STATEMENT Optimizer=ALL_ROWS
1	0	NESTED LOOPS
2	1	  TABLE ACCESS (BY INDEX ROWID) OF '사원' (TABLE)
3	2       INDEX (RANGE SCAN) OF '사원_X1' (INDEX)
4	3     TABLE ACCESS (BY INDEX ROWID) OF '고객' (TABLE)
5	4       INDEX (RANGE SCAN) OF '고객_X1' (INDEX)
```

- 위쪽테이블 (사원) = Outer Table = Driving Table
- 아래테이블 (고객) = Inner Table

#### NL조인 힌트 기술하는 방법

1. use_nl 힌트

```sql
select /*+ use_nl(A, B, C, D) */ *
from A, B, C, D
where ...
```

- `use_nl` : 네 개 테이블을 NL방식으로 조인해라 
- `(A, B, C, D)` : 순서는 옵티마이저가 스스로 정하도록 맡긴 것이다.

2. ordered 힌트

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

3. leading 힌트

```sql
select /*+ leading(A, B, C, D) use_nl(A) use_nl(D) use_hash(B) */ *
from A, B, C, D
where ...
```

- `leading` : FROM절 순서에 관계 없이, leading 뒤에 기술한 순서대로 조인하라고 옵티마이저에게 지시하는 힌트


### 4.1.3 NL 조인 수행 과정 분석

### 4.1.4 NL 조인 튜닝 포인트

### 4.1.5 NL 조인 특징 요약

### 4.1.6 NL 조인 튜닝 실습

### 4.1.7 NL 조인 확장 매커니즘
