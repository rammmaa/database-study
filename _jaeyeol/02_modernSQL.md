---
title: 02 Modern SQL
author: jaeyeol
layout: post
---

# Early
1971: SQUARE (너무 복잡함)  
1972: SEQUEL  
1980년대: commercial DBMS 등장, SEQUEL 대신 SQL으로 이름이 바뀜  
최신 SQL 표준: SQL:2023  
갖추어야 할 최소한의 syntax가 갖춰진 가장 낮은 버전: SQL-92  
Relational Algebra는 집합 연산이라 중복을 허용하지 않는 반면 SQL의 경우 `std::multiset`처럼 중복을 허용함  
세 가지 기본 구성요소: DDL (Schema 정의), DML (SELECT, INSERT, UPDATE, ...), DCL (사용자 권한 제어)  

# Aggregate
여러 데이터를 한 번에 뭉탱이로 처리하는 것들  
AVG, MIN, MAX, SUM, COUNT 등의 aggregate function 존재  
GROUP BY: 특정 column 값 기준으로 값들을 partition함  
GROUPING SETS: GROUP BY에서 aggregate를 좀 더 자세하게 할 수 있는 버전  
HAVING: GROUP BY로 묶인 것들 사이에서 WHERE과 비슷한 것

# String Operations
DBMS마다 case-sensitivity, 큰따옴표/작은따옴표 등 디테일이 다르므로 주의가 필요함  
LIKE: 와일드카드 매칭, _ = Σ, % = Σ*  
SIMILAR TO: Regular Expression 매칭  
||: 문자열 여러 개 합치기  
이외에도 SUBSTRING, UPPER, LOWER 등의 함수 존재

# Datetime Operations
Datetime의 경우 DBMS마다 문법이 다 다르기 때문에 간단히 정리할 수 없음  
보통 NOW, CURRENT_TIMESTAMP 정도는 사용 가능하지만 이것도 값으로 정의되어 있는지 함수로 정의되어 있는지 다르므로 주의 필요

# Output Control
ORDER BY: 특정 column 순으로 값 정렬  
FETCH, OFFSET: 결과의 a번째 줄부터 b번째 줄까지 처럼 특정 구간만 추출해낼 수 있음

# Output Redirection
INTO: 새 테이블을 하나 만들어서 쿼리의 결과를 그곳에 저장

# Nested Queries
정의: SELECT 쿼리 어딘가에 또다른 SELECT 구문을 넣는 것  
ALL, ANY(또는 IN), EXISTS: 포함 관계 확인하는 구문  
LATERAL: 2중 for문처럼 동작해서 LATERAL 앞에서 정의한 걸 참조할 수 있게 해 줌

# CTE (Common Table Expression)
result set을 임시로 하나 만들어서 그게 원래부터 있던 테이블인 것처럼 쓸 수 있음, WITH ~ AS로 사용

# Window Function
moving average처럼 뭔가 순서가 중요한 작업을 하고 싶을 때 사용하면 좋음  
SELECT FUNC(...) OVER (...)의 문법으로 사용 가능  
FUNC 자리에는 aggregation function, ROW_NUMBER(), RANK() 등의 함수가 들어갈 수 있음  
출력 결과는 Aggregation과 비슷하지만 GROUP BY와는 다르게 값들을 묶지는 않음  
GROUP BY와 비슷한 걸 쓰고 싶으면 PARTITION BY를 사용하면 됨
