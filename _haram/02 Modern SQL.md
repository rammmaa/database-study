---
title: 02 Modern SQL
author: haram
layout: post
---
## 1 SQL History

- SQL:1999 정규표현식, 트리거
- SQL:2003 XML, 윈도우, 시퀀스
- SQL:2008 잘라내기(Truncation), 고급 정렬(Fancy Sorting)
- SQL:2011 시간(Temporal) DB, 파이프라인 DML
- SQL:2016 JSON, 다형 테이블(Polymorphic tables)
- SQL:2023 프로퍼티 그래프 쿼리, 다차원 배열

* 어떤 시스템이 “SQL을 지원한다”고 주장하려면 최소한 **SQL-92**의 문법을 지원해야 한다. 각 벤더는 표준을 일정 수준 따르지만, 독자적인 확장(proprietary extensions)도 많다.

## 2 Relational Languages

* SQL은 여러 종류의 명령(클래스)로 구성된다.
* 1) **DML (Data Manipulation Language)**: `SELECT`, `INSERT`, `UPDATE`, `DELETE`
* 2) **DDL (Data Definition Language)**: 테이블, 인덱스, 뷰 등 객체의 스키마 정의
* 3) **DCL (Data Control Language)**: 보안/접근 제어
* 4) 그 외에도 뷰 정의, 무결성 및 참조 무결성 제약, 트랜잭션을 포함한다.
* SQL의 기반이 되는 관계대수(relational algebra)는 **집합(set)**(순서 없는 컬렉션, 중복 불가)을 사용한다. 하지만 SQL은 기본적으로 중복 제거 작업을 피하기 위해 **가방/멀티셋(bag)**(순서 없는 컬렉션, 중복 허용)을 기반으로 한다. 원한다면 DISTINCT와 같은 기능으로 중복을 제거할 수 있다.

## 3 Example Database

```sql
CREATE TABLE student (
  sid INT PRIMARY KEY,
  name VARCHAR(16),
  login VARCHAR(32) UNIQUE,
  age SMALLINT,
  gpa FLOAT
);

CREATE TABLE course (
  cid VARCHAR(32) PRIMARY KEY,
  name VARCHAR(32) NOT NULL
);

CREATE TABLE enrolled (
  sid INT REFERENCES student (sid),
  cid VARCHAR(32) REFERENCES course (cid),
  grade CHAR(1)
);

```

## 4 Aggregates
* 집계 함수(aggregation function)는 입력으로 튜플의 bag(중복 허용 컬렉션)을 받고, 출력으로 단일 스칼라 값을 만든다. 집계 함수는 (거의) `SELECT`의 출력 리스트에서만 사용할 수 있다.
	* `AVG(COL)`: COL 값의 평균
	- `MIN(COL)`: COL 값의 최솟값
	- `MAX(COL)`: COL 값의 최댓값
	- `SUM(COL)`: COL 값의 합
	- `COUNT(COL)`: 관계(테이블)의 튜플 개수
	
* 아래 세 쿼리는 동일하다.
```sql
SELECT COUNT(*) FROM student WHERE login LIKE '%@cs';
SELECT COUNT(login) FROM student WHERE login LIKE '%@cs';
SELECT COUNT(1) FROM student WHERE login LIKE '%@cs';
```

* '@cs' 로그인인 학생들의 고유 학생 수 
```sql
SELECT COUNT(DISTINCT login)
FROM student WHERE login LIKE '%@cs';
```

* ‘@cs’ 로그인인 학생 수와 평균 GPA
```sql
SELECT AVG(gpa), COUNT(sid)
FROM student WHERE login LIKE '%@cs';
```
* 집계가 아닌 컬럼을 집계와 함께 출력하면(아래 예시의 `e.cid`) 그 값은 **정의되지 않는다**. 대부분의 현실 DBMS는 이 경우 에러를 내지만, SQLite 같은 일부 시스템은 임의 값을 골라 허용한다. SQL:2023 표준에는 이를 명시적으로 수행하는 `ANY_VALUE` 집계 함수가 추가되었다.

* 각 course별 평균 GPA (잘못된/정의되지 않는 형태)

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e JOIN student AS s
ON e.sid = s.sid;
```

* 각 course별 평균 GPA (`ANY_VALUE` 사용)

```sql
SELECT AVG(s.gpa), ANY_VALUE(e.cid)
FROM enrolled AS e JOIN student AS s
ON e.sid = s.sid;
```

* `SELECT` 출력 절에서 **집계되지 않은 값**은 반드시 `GROUP BY` 절에 등장해야 한다. `GROUP BY`는 튜플들을 값에 따라 분할(partition)하고, 각 부분집합마다 집계를 계산한다.

* `GROUP BY`로 각 course별 평균 GPA

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e JOIN student AS s
WHERE e.sid = s.sid
GROUP BY e.cid;
```

* `GROUPING SETS`는 여러 가지 그룹핑을 한 쿼리에서 지정할 수 있게 해준다. 여러 쿼리를 `UNION`으로 묶는 대신, DBMS가 데이터를 한 번만 스캔하면 되도록 할 수 있다.

* (course, grade)별 학생 수 + course별 학생 수 + 전체 학생 수

```sql
SELECT c.name AS c_name, e.grade,
       COUNT(*) AS num_students
FROM enrolled AS e
JOIN course AS c ON e.cid = c.cid
GROUP BY GROUPING SETS (
  (c.name, e.grade),
  (c.name),
  (),
);
```

## HAVING 절

`HAVING` 절은 **집계 결과를 기준으로 출력 결과를 필터링**한다, 반면 `WHERE`는 “행(row)”을 필터링한다. 그래서 `HAVING`은 `GROUP BY`에 대한 `WHERE`처럼 동작한다.

* 평균 GPA가 3.9보다 큰 course들의 집합

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING avg_gpa > 3.9;
```

* 위 예시는 구식의 **암시적 조인(implicit join)** 문법을 사용한다(WHERE 절 처리를 위해 조인이 필요함을 DBMS가 추론). 실제로는 항상 **명시적 조인(explicit join)**을 쓰는 게 좋다.
* 또한 위와 같은 “별칭(avg_gpa)을 HAVING에서 재사용”하는 문법은 많은 DBMS가 지원하지만 SQL 표준에는 부합하지 않는다. 표준에 맞추려면 HAVING에서 `AVG(s.gpa)`를 다시 써야 한다.

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING AVG(s.gpa) > 3.9;
```

---

## 5 String Operations (문자열 연산)

SQL 표준은 문자열이 **대소문자 구분**이며, 홑따옴표(single-quote)만 사용한다고 말한다. 하지만 실제 시스템(예: MySQL)은 이 부분에서 허용 범위가 다를 수 있다.

쿼리의 어느 부분에서든 사용할 수 있는 문자열 조작 함수들이 있다.

- **패턴 매칭(Pattern Matching)**: `LIKE` 키워드를 사용
    - `%` : (빈 문자열 포함) 임의의 길이 문자열과 매칭
    - `_` : 임의의 한 글자와 매칭
        
- **정규식 매칭**: `SIMILAR TO`는 정규식 매칭을 허용하지만, 모든 시스템에서 지원되지는 않는다. (각자 다른 문법을 쓰는 경우가 많음).
    
- **문자열 함수**: SQL-92는 문자열 함수를 정의하며, 많은 DBMS는 표준 외 함수도 추가로 제공한다. 표준 함수 예로 `SUBSTRING(S, B, E)`, `UPPER(S)` 등이 있다.
    
- **연결(Concatenation)**: 세로 막대 두 개 `||`는 문자열들을 이어붙여 하나의 문자열로 만든다(시스템마다 다른 기호를 쓰기도 함).
    
---

## 6 Date and Time (날짜/시간)

* DB는 날짜와 시간을 추적해야 하는 경우가 많아서, SQL은 `DATE`, `TIME` 속성을 조작하는 연산을 지원한다. 이는 출력으로도, 조건(predicate)으로도 사용할 수 있다.
* 다만 날짜/시간 연산의 구체 문법은 시스템마다 크게 다를 수 있다.

---

## 7 Output Redirection (출력 리다이렉션)

쿼리 결과를 클라이언트(예: 터미널)로 반환하는 대신, DBMS가 결과를 **다른 테이블에 저장**하도록 할 수 있다. 이후 쿼리에서 그 데이터를 사용할 수 있다.

- **새 테이블(New Table)**: 쿼리 출력 결과를 새로운(영구) 테이블에 저장  

```sql
SELECT DISTINCT cid INTO CourseIds FROM enrolled;
```

- **기존 테이블(Existing Table)**: 이미 존재하는 테이블에 저장  
    대상 테이블은 출력과 **컬럼 개수 및 타입이 같아야** 하지만, 출력 쿼리의 컬럼 이름은 반드시 일치할 필요는 없다.

```sql
INSERT INTO CourseIds (SELECT DISTINCT cid FROM enrolled);
```

---

## 8 Output Control (출력 제어)

SQL 결과는 **순서가 없다(unordered)**. 따라서 튜플에 정렬을 부여하려면 `ORDER BY` 절을 사용해야 한다.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
ORDER BY grade;
```

기본 정렬은 오름차순(`ASC`)이며, 내림차순은 `DESC`로 지정한다.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
ORDER BY grade DESC;
```

여러 `ORDER BY` 키를 사용해 타이브레이크나 복잡한 정렬을 할 수 있다.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
ORDER BY grade DESC, sid ASC;
```

`ORDER BY`에는 임의의 표현식을 사용할 수도 있다.

```sql
SELECT sid FROM enrolled WHERE cid = '15-721'
ORDER BY UPPER(grade) DESC, sid + 1 ASC;
```

기본적으로 DBMS는 쿼리가 만든 모든 튜플을 반환한다. 많은 시스템이 “처음 N개만” 가져오는 자체 문법을 제공하지만, 흔히 쓰는 것은 `LIMIT` 절이다.

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
LIMIT 10;
```

또한 `OFFSET`을 줘서 결과 범위를 지정할 수 있다.

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
LIMIT 20 OFFSET 10;
```

* `ORDER BY` 없이 `LIMIT`만 쓰면, 관계 모델은 순서를 강제하지 않기 때문에 DBMS 내부 사정에 따라 쿼리를 실행할 때마다 다른 튜플이 결과로 나올 수 있다.
* 또한 SQL은 `INTO` 키워드를 통해 쿼리 결과를 다른 테이블로 저장할 수 있으며, 어떤 시스템은 `INTO TEMPORARY`로 임시 테이블로의 리다이렉션도 허용한다.

---

## 9 Window Functions

윈도우 함수(window function)는 서로 관련 있는 튜플 집합에 대해 “슬라이딩” 계산을 수행한다. 집계 함수와 비슷하지만, 튜플들이 하나의 출력 튜플로 **접히지(collapsed) 않는다**.


1. 테이블을 파티션(partition)으로 나눈다    
2. 각 파티션을 정렬한다(`ORDER BY`가 있을 때)
3. 각 레코드에 대해 여러 레코드를 span하는 “윈도”를 만든다
4. 각 윈도에 대해 출력 값을 계산한다
    
    

`OVER` 절은 윈도우 함수를 계산할 때 튜플을 어떻게 묶을지 지정한다. 그룹 지정은 `PARTITION BY`로 한다.

```sql
SELECT cid, sid, ROW_NUMBER() OVER (PARTITION BY cid)
FROM enrolled ORDER BY cid;
```

또한 `OVER` 안에 `ORDER BY`를 넣어 DB가 내부적으로 바뀌어도 결과 순서가 결정적(deterministic)으로 나오게 할 수 있다.

```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY cid)
FROM enrolled ORDER BY cid;
```

중요: DBMS는 윈도우 함수 정렬 이후에 `RANK`를 계산하지만, `ROW_NUMBER`는 정렬 전에 계산한다.

* 각 course에서 두 번째로 높은 성적을 가진 학생 찾기
* 성적이 숫자가 아니라 A,B,C이므로 `ASC`로 정렬함.

```sql
SELECT * FROM (
  SELECT *, RANK() OVER (PARTITION BY cid
                         ORDER BY grade ASC) AS rank
  FROM enrolled
) AS ranking
WHERE ranking.rank = 2;
```

---

## 10 Nested Queries (중첩 질의)

* 중첩 질의는 다른 쿼리 안에 쿼리를 넣어, 한 쿼리에서 더 복잡한 로직을 실행하게 한다. 중첩 질의는 최적화가 어려운 경우가 많다. 바깥(outer) 쿼리의 스코프(scope)는 안쪽(inner) 쿼리에 포함된다(즉 inner가 outer의 속성에 접근 가능).
* 반대는 성립하지 않는다.

1. `SELECT` 출력 타깃

```sql
SELECT (SELECT 1) AS one FROM student;
```

2. `FROM` 절
   
```sql
SELECT name
FROM student AS s, (SELECT sid FROM enrolled) AS e
WHERE s.sid = e.sid;
```

3. `WHERE` 절

```sql
SELECT name FROM student
WHERE sid IN ( SELECT sid FROM enrolled );
```

* ‘15-445’에 수강 등록된 학생 이름 구하기

```sql
SELECT name FROM student
WHERE sid IN (
  SELECT sid FROM enrolled
  WHERE cid = '15-445'
);
```


* 적어도 한 과목 이상 수강 중인 학생들 중, 가장 큰 id(sid)를 가진 학생 레코드 찾기

```sql
SELECT student.sid, name
FROM student
JOIN (SELECT MAX(sid) AS sid
      FROM enrolled) AS max_e
ON student.sid = max_e.sid;
```

중첩 질의 결과에 대해 사용할 수 있는 표현

- `ALL`: 서브쿼리의 모든 행에 대해 표현식을 만족해야 함
- `ANY`: 서브쿼리의 적어도 한 행에 대해 표현식을 만족하면 됨
- `IN`: `= ANY()`와 동등
- `EXISTS`: 최소 한 행이 반환되면 참

* 수강생이 한 명도 없는 course 찾기

```sql
SELECT * FROM course
WHERE NOT EXISTS(
  SELECT * FROM enrolled
  WHERE course.cid = enrolled.cid
);
```

---

## 11 Lateral Joins

`LATERAL` 연산자는 어떤 중첩 쿼리가 자신보다 앞에 있는 다른 중첩 쿼리의 속성을 참조할 수 있게 해준다. 레터럴 조인은 테이블의 각 튜플마다 다른 쿼리를 호출하는 for 루프처럼 생각할 수 있다.

* 각 course마다 (1) 수강생 수, (2) 수강생들의 평균 GPA를 계산하고, 수강생 수 기준 내림차순 정렬
* course 레코드를 얻은 뒤 각 course에 대해
	* 이 course의 수강생 수 계산
	* 이 course 수강생들의 평균 GPA 계산
    
```sql
SELECT * FROM course AS c
LATERAL (SELECT COUNT(*) AS cnt FROM enrolled
         WHERE enrolled.cid = c.cid) AS t1,
LATERAL (SELECT AVG(gpa) AS avg FROM student AS s
         JOIN enrolled AS e ON s.sid = e.sid
         WHERE e.cid = c.cid) AS t2;
```

---

## 12 Common Table Expressions (CTE, 공통 테이블 식)

* CTE(Common Table Expression)는 더 복잡한 쿼리를 작성할 때 **윈도 함수나 중첩 질의의 대안**이 될 수 있다. 더 큰 쿼리에서 사용할 보조(auxiliary) 문장을 작성할 수 있게 하며, 하나의 쿼리 범위에 한정된 임시 테이블로 생각할 수 있다.
* `WITH` 절은 내부 쿼리의 출력 결과를 같은 이름의 임시 테이블에 바인딩한다.

* cteName이라는 CTE(값 1 하나만 가진 튜플)를 만들고, cteName의 모든 속성 선택

```sql
WITH cteName AS (
  SELECT 1
)
SELECT * FROM cteName;
```

* `AS` 앞에서 출력 컬럼 이름을 바인딩할 수도 있다.

```sql
WITH cteName (col1, col2) AS (
  SELECT 1, 2
)
SELECT col1 + col2 FROM cteName;
```

* 하나의 쿼리에 여러 CTE 선언을 포함할 수 있다.

```sql
WITH cte1 (col1) AS (SELECT 1),
     cte2 (col2) AS (SELECT 2)
SELECT * FROM cte1, cte2;
```

* `WITH` 뒤에 `RECURSIVE` 키워드를 붙이면 CTE가 자기 자신을 참조할 수 있고, 이를 통해 SQL 쿼리에서 재귀를 구현할 수 있다. 재귀 CTE를 사용하면 SQL은 (다소 번거롭다는 점을 제외하면) 계산 표현력 면에서 범용 프로그래밍 언어만큼 강력하다는 의미로, **튜링 완전(Turing-complete)임이 증명된다.

* 1부터 10까지의 수열 출력

```sql
WITH RECURSIVE cteSource (counter) AS (
  ( SELECT 1 )
  UNION
  ( SELECT counter + 1 FROM cteSource
    WHERE counter < 10 )
)
SELECT * FROM cteSource;
```
