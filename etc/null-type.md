데이터베이스마다 NULL을 처리하는 방식이 미세하게 다르기 때문에, 멀티 DB 환경에서 작업할 때 헷갈리기 쉽습니다. 
Oracle, MySQL, MSSQL, PostgreSQL 4대 천왕을 중심으로 한눈에 볼 수 있게 비교 요약해 드립니다.

### 📊 주요 DBMS별 NULL 처리 비교표

| 구분 | Oracle | MySQL | MSSQL (SQL Server) | PostgreSQL |
|---|---|---|---|---|
| 빈 문자열 ('') | NULL로 취급 | 데이터로 취급 (NULL 아님) | 데이터로 취급 (NULL 아님) | 데이터로 취급 (NULL 아님) |
| 기본 치환 함수 | NVL | IFNULL | ISNULL | - (표준 함수 사용) |
| 표준 치환 함수 | COALESCE | COALESCE | COALESCE | COALESCE |
| 조건부 치환 | NVL2 | IF | - | - |
| 정렬 시 기본 위치 | 마지막 (가장 큼) | 처음 (가장 작음) | 처음 (가장 작음) | 마지막 (가장 큼) |
| 정렬 옵션 지원 | NULLS FIRST/LAST | 편법 사용 필요* | - | NULLS FIRST/LAST |
| 문자열 연결 연산 | A || NULL = A | CONCAT(A, NULL) = NULL | A + NULL = NULL | A || NULL = NULL |

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
