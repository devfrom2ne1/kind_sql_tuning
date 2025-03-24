# 2. 인덱스 기본

## 2.2 인덱스 기본 사용법

### 2.2.1 인덱스를 사용한다는 것

- 인덱스 컬럼을 가공하면 인덱스를 정상적으로 사용할 수 없다. 
- 인덱스 스캔 시작점을 찾을 수 없게 되기 때문이다.

### 2.2.2 인덱스를 Range Scan 할 수 없는 이유

#### between 조건절

```sql
where birth_dt between '20070101' and '20070131'
```

- 예를 들어, 생년월일 순으로 학생을 줄세웠다고 생각해보자.
- 이때 2007년 1월 생을 찾는다면, 2007년 1월 1일에 태어난 학생부터 시작해서 2007년 2월 1일에 태어난 학생 전까지 탐색하면 된다. 

#### substr 조건절

```sql
where substr(birth_dt, 5, 2) = '05'
```

- 하지만, 년도에 관계없이 5월생을 찾는다면?
- 이렇게 되면 시작점을 잡을 수 가 없어서, 인덱스를 Range Scan 할 수 없게 된다. 

#### LIKE 조건절

```sql
where 업체명 like '대한%'
```

- 특정 문자열로 시작하는 단어는 특정 구간에 모여있어서 Range Scan이 가능하다.


```sql
where 업체명 like '%대한%'
```

- 하지만, 중간에 있는 값들은 전 구간에 걸쳐있어서 Range Scan을 할 수 없다.


#### OR 조건절

```sql
where (전화번호 = :tel_no OR 고객명 = :cust_nm)
```

- 수직적 방식으로 시작점을 찾을 수 없고, 어떤 방식으로 인덱스를 구성해도 Range Scan이 불가능하다.

```sql
select * 
from 고객
where 고객명 = :cust_nm
union all
select * 
from 고객
where 전화번호 = :tel_no
and (고객명 <> :cust_nm or 고객명 is null)
```

- 하지만 이렇게 옵티마이저가 `OR Expansion` 형태로 변환할 수 있다. 
- 이 때는 Index Range Scan이 발동했다. 

#### IN 조건절

- IN은 OR나 같다. 
- 그래서 `UNION ALL` 방식으로 작성하면 브랜치 별로 인덱스 스캔 시작점을 알 수 있어 Range Scan이 가능하다.
- 아니면 자동으로 SQL 옵티마이저가 IN-List Iterator 방식을 사용해서 List 갯수만큼 Index Range Scan을 반복한다. 

### 2.2.3 더 중요한 인덱스 사용 조건

- 인덱스 선두 컬럼이 조건절에 가공하지 않은 상태로 있어야 한다!
- 하지만 Range Scan을 한다고 해도, 인덱스 리프 블록에서 스캔하는 양을 따져보면 성능이 안 좋은 경우도 있다. 
- 인덱스를 타도 스캔 범위가 너무 넓기 때문이다. 

### 2.2.4 인덱스를 이용한 소트 연산 생략

- 인덱스는 정렬되어 있다. 
- 그래서 SQL에 ORDER BY 가 있어도 실행계획에선 SORT ORDER BY 없이 실행된다. 
- `INDEX (RANGE SCAN DESCENDING) OF ...` 를 보면 인덱스 자체로 내림차순, 오름차순을 시행함을 알 수 있다. 

### 2.2.5 ORDER BY 절에서 컬럼 가공

- ORDER BY에 쓰인 컬럼명은 테이블 컬럼명이 아닌 SELECT-LIST에 적힌 컬럼명이다. 
- 그래서 SELECT-LIST에서 가공된 컬럼명이 Alias로 적혀 있으면, 해당 가공된 컬럼을 기준으로 SORT ORDER BY 연산이 실행계획에 나타난다. 

* 친절한 SQL튜닝 98~99페이지 참고하기

### 2.2.6 SELECT-LIST에서 컬럼 가공

- 인덱스를 이용해 최댓값 또는 최소값을 구할때는, 인덱스 리프 블록의 왼쪽(MIN) 또는 오른쪽(MAX)에서 레코드 하나(FIRST ROW)만 읽고 멈춘다. 

### 2.2.7 자동 형변환

```sql
select * from cust_base
where 생년월일 = 19881225
```

생년월일 컬럼이 문자열인데, 조건절 비교값을 숫자형으로 표현한다면 오라클 DBMS는 자동으로 형변환 처리를 해준다. 


```sql
select * from cust_base
where TO_NUMBER(생년월일) = 19881225
```

- 숫자형 vs 문자형 
    - 숫자형 기준으로 문자형 컬럼을 변환한다. 
    - LIKE 연산자는 반대이다! LIKE가 문자열 비교 연산자이기 때문이다. 

- 날짜형 vs 문자형
    - 날짜형 기준으로 문자형 컬럼을 변환한다. 

- decode 형변환 주의
    - decode(a, b, c, d) 에서 c의 데이터타입을 기준으로 반환값의 데이터 타입이 결정된다. 
    - 세 번째 인자가 null 이라면 varchar2로 취급된다. 
    - decode(job, 'PRESIDENT', to_number(NULL), sal) 로 NULL의 데이터타입을 명시해줘야 한다. 
