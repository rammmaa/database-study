---
title: 02 Modern SQL
author: haram
layout: post
---

~역사부분 흥미롭지만 적지않을게요 그래도 잘 들었어요 교수님^^;;~

* SQL은 무엇을 원한다고 쓰는 선언형 언어. 따라서 How가 아니라 What으로 **선언**함
* IBM System R의 SQUARE → SEQUEL → SQL
* 관계대수는 set(중복 없음), SQL은 bag(중복 허용) 기반. 필요하면 필요하면 DISTINCT로 제거 가능

## 집계(Aggregation), GROUP BY, GROUPING SETS, HAVING

* AVG/MIN/MAX/SUM/COUNT
* COUNT(\*), COUNT(col), COUNT(1)은 케이스에 따라 차이가 생길 수 있고(특히 NULL), 관례적으로 COUNT(\*)를 많이 씀. COUNT(DISTINCT col) 같은 형태도 자주 사용
* SELECT에 집계가 아닌 컬럼을 같이 내보내려면 그 컬럼은 GROUP BY에 포함되어야 함 > 집계 + 비집계 컬럼을 같이 SELECT하면 결과가 정의되지 않음

### GROUPING SETS

* 여러 가지 그룹 기준을 한 번에 만들 때 유용
* UNION ALL로 여러 쿼리를 붙이는 대신, 엔진이 더 효율적으로 처리할 수 있음

## String / Date / Time

### String

* 문자열 리터럴은 '' 사용
* 패턴 매칭: LIKE (%, _)

## Output Control

### ORDER BY
* 정렬 기준 명시 가능

### Top-N / Paging
* LIMIT/OFFSET, FETCH FIRST ..., TOP
* 결과를 테이블로 저장

### Subqueries