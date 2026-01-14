데이터베이스마다 NULL을 처리하는 방식이 미세하게 다르기 때문에, 멀티 DB 환경에서 작업할 때 헷갈리기 쉽습니다. 

### 📊 주요 DBMS별 NULL 처리 비교표

| 구분 | Oracle | MySQL | MSSQL (SQL Server) | PostgreSQL |
|---|---|---|---|---|
| 치환 함수 | NVL(col, 0) | IFNULL(col, 0) | ISNULL(col, 0) | COALESCE(col, 0) |
| 다중 치환 | COALESCE | COALESCE | COALESCE | COALESCE |
| 빈 문자열 ('') | NULL과 동일함 | 데이터로 인식 | 데이터로 인식 | 데이터로 인식 |
| 정렬(ASC) 시 | 맨 뒤 (가장 큼) | 맨 앞 (가장 작음) | 맨 앞 (가장 작음) | 맨 뒤 (가장 큼) |
| 문자열 결합 | `A || NULL` = A | CONCAT 사용 시 NULL | `A + NULL` = NULL | `A || NULL` = NULL |
| 정렬 옵션 | NULLS FIRST/LAST | 편법(IS NULL) 사용 | 지원 안 함 | NULLS FIRST/LAST |

### 🔍 핵심 차이점 상세 분석
#### 1. 빈 문자열('')에 대한 태도
- Oracle: INSERT INTO table (col) VALUES (''); 하면 col에는 NULL이 들어갑니다.
- 나머지 (MySQL, MSSQL, PG): 빈 문자열도 엄연한 '길이가 0인 데이터'로 봅니다. IS NULL로 검색하면 조회되지 않습니다.

#### 2. 문자열 결합 (Concatenation)
> Tip: CONCAT_WS (MySQL, PG)나 ISNULL/COALESCE를 사용해 방어 코드를 짜야 합니다.

- Oracle: NULL을 무시하고 연결합니다. ('Hello' || NULL → 'Hello')
- MySQL/MSSQL/PG: 하나라도 NULL이면 전체 결과가 NULL이 됩니다. 
	
#### 3. 정렬 순서 (ORDER BY)
- 데이터를 오름차순(ASC)으로 정렬했을 때
	- Oracle / PostgreSQL: NULL은 **끝(Bottom)**에 나옵니다.
	- MySQL / MSSQL: NULL은 **맨 위(Top)**에 나옵니다.

> MySQL에서 NULL을 뒤로 보내고 싶다면?
> ORDER BY 컬럼 IS NULL ASC, 컬럼 ASC 같은 방식으로 수동 지정해야 합니다.

### 💡 각 DB별 치환 함수 예시 코드
#### 1. Oracle

```sql
SELECT NVL(comm, 0), NVL2(comm, '있음', '없음') FROM emp;
```

#### 2. MySQL

```sql
SELECT IFNULL(comm, 0), IF(comm IS NULL, '없음', '있음') FROM emp;
```

#### 3. MSSQL

```sql
SELECT ISNULL(comm, 0) FROM emp;
```

#### 4. 공통 (PostgreSQL 및 모든 DB)
-- 가급적 표준인 COALESCE를 쓰는 것이 DB 이관 시 유리합니다.

```sql
SELECT COALESCE(comm, 0) FROM emp;
```

### 집계 연산 (공통 사항)
모든 주요 DB에서 집계 함수는 아래와 같이 동작합니다.
- COUNT(*) : 테이블의 물리적인 행(Row) 개수를 셉니다. NULL을 포함합니다.
- COUNT(컬럼명) : 해당 컬럼에 값이 있는 행만 셉니다. NULL은 제외됩니다.
- SUM, AVG, MAX, MIN : 모두 NULL을 무시하고 연산합니다.

```
	 데이터가 {10, NULL, 20} 일 때:
     * SUM = 30
     * AVG = 15 (30 나누기 2)
     * MAX = 20
```

### 집합 연산 (Union / Union All)
- UNION (중복 제거):
	- 두 집합에 모두 NULL이 있으면, 이를 같은 값으로 중복 처리하여 최종 결과에는 하나의 NULL만 남깁니다.
- UNION ALL (전체 합계):
	- 중복을 따지지 않으므로 양쪽의 NULL이 모두 결과에 포함됩니다.


###  DB별 특이점 및 꿀팁
🔹 Oracle
 * 빈 문자열 주의: INSERT INTO tab (col) VALUES ('');를 실행하면 col에는 NULL이 들어갑니다. 따라서 WHERE col = ''로는 조회가 안 되고 반드시 WHERE col IS NULL을 써야 합니다.
 * NVL2: NVL2(컬럼, '값있음', '값없음') 함수가 있어 매우 편리합니다.
🔹 MySQL
 * IFNULL vs COALESCE: 둘 다 사용 가능하지만, 여러 인자를 체크할 때는 표준인 COALESCE를 권장합니다.
 * 정렬 꼼수: ORDER BY 컬럼 ASC 시 NULL을 뒤로 보내고 싶다면 ORDER BY -컬럼 DESC 또는 ORDER BY 컬럼 IS NULL, 컬럼 ASC를 사용합니다.
🔹 MSSQL
 * SET CONCAT_NULL_YIELDS_NULL: 이 옵션 설정에 따라 문자열 결합 시 NULL 처리 방식이 달라질 수 있지만, 기본적으로는 NULL과 합치면 NULL이 되는 것이 원칙입니다.
 * ISNULL: Oracle의 NVL과 이름이 비슷하지만 MSSQL 전용입니다.
🔹 PostgreSQL
 * 표준에 가장 엄격: NVL이나 IFNULL 같은 전용 함수보다는 표준인 COALESCE를 적극적으로 사용합니다.
 * Boolean 타입: NULL은 TRUE도 FALSE도 아닌 UNKNOWN 상태를 가집니다.


### 실전 예시 

```sql
-- [공통] NULL을 0으로 바꾸어 합계 구하기
SELECT SUM(COALESCE(score, 0)) FROM exams;

-- [Oracle] 보너스가 있으면 급여+보너스, 없으면 급여만
SELECT salary + NVL(bonus, 0) FROM employees;

-- [MySQL] NULL을 맨 뒤로 보내며 정렬
SELECT * FROM users ORDER BY nickname IS NULL ASC, nickname ASC;

-- [MSSQL] NULL이면 'N/A' 출력
SELECT ISNULL(phone_number, 'N/A') FROM customers;
```