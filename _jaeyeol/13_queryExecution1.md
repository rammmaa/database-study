---
title: 13 Query Execution I
author: jaeyeol
layout: post
---

13, 14강에서 알아볼 것은 Query Execution
13강은 single-threaded, 14강은 multi-threaded 환경을 가정하는 것 같음

# Query Plan
지난 몇 개의 강의에서 query plan이 지속적으로 나왔었는데, 이번에 이걸 제대로 팔 예정
복습해 보자면, Query Plan은 "operator"들로 이루어진 DAG
여기에서 여러 operator 간에 데이터가 "연속적으로 흐를" 수 있는 것들을 pipeline이라고 정의
이와 반대로, 자식노드가 뱉어준 값을 모두 알아야만 값을 뱉을 수 있는 것들을 pipeline breaker이라고 정의 (join, sort, subquery, ...)

# Processing Model
DBMS가 어떻게 query plan을 실행하는지 등을 정의
자연스럽게 OLTP, OLAP workload 중 뭐에 더 맞추는지에 대한 tradeoff 발생
각 모델은, control flow(어떻게 operator 실행할 것인가?), data flow(어떻게 결과를 보낼 것인가?) 가 존재
output의 경우, 튜플 전체이거나(NSM), column들의 부분집합(DSM)이 될 수 있음

## Iterator Model
Volcano Model, Pipeline Model이라고도 불림
각각의 operator에게는 Next()라는 함수가 존재
부모노드가 Next()를 호출하면 자식노드가 다음 tuple을 반환하는 식
Next()함수는 C++같은 곳에 있는 iterator 개념과 비슷하다고 생각하면 될 듯함
pipelining도 가능해서, 다음 tuple을 가져오기 전에 현재 보고 있는 tuple을 query plan tree 위로 최대한 올릴 수 있음

## Materialization Model
각각의 operator는 모든 데이터를 한 번에 입출력함
Iterator Model과는 다르게 LIMIT절 등을 처리하는 데 하자가 있을 수 있음

### Operator Fusion
예를 들어 아래에서 10억개의 tuple을 가지고 올라왔는데 위에 있는 WHERE절이 10개만 원할 때?
사실 10개만 들고 왔어도 되는데 10억개씩이나 메모리로 가져온 셈이 되므로 매우 비효율적
그러므로 WHERE절을 데이터를 가져오는 거랑 합쳐버리는 식으로 해결 가능

OLTP에 최적화되어 있으며 (함수 호출 횟수 적음), OLAP에는 별로 (buffer 많이 필요할 수도 있음)

## Vectorization Model
Next() 함수가 존재하는데 이번에는 operator가 tuple의 뭉탱이를 뱉음
한 뭉탱이가 몇 개 tuple일지는 implementation-defined인 듯함
buffer을 극도로 많이 요하지 않으면서 함수 호출 횟수를 줄이므로 OLAP에 매우 최적화됨

# Plan Processing Direction

## Top-to-Bottom (Pull-based)
지금까지 말한 것은 모두 이 방법을 assume했음
특수한 경우가 아닌 이상 tuple을 전달할 때 함수 호출로 전달함
처음에 루트 노드에서 시작해, 루트가 Next()를 부르면 아래 노드로 가고, 그런 식

## Bottom-to-Up (Push-based)
Next()함수와 같은 게 존재하지 않음
반대로 리프 노드에서 시작해 부모 노드로 데이터를 'push'함
pipeline 하나를 통째로 합쳐버릴 수도 있음

좀 더 흔한 건 top-to-bottom 쪽임
Note that, Processing Model과 Plan Processing Direction은 독립적인 개념

# Access Methods
DBMS가 테이블에 있는 데이터를 어떻게 가져올 것인가?
relational algebra에서는 데이터를 가져오라고만 되어 있지, '어떻게'에 대해서는 정의된 바가 없음

## Sequential Scan
page를 sequential scan하며 데이터를 읽어오고, 뱉든지 말든지 하는 방법
DBMS는 내부적으로 cursor을 만들어 가장 마지막으로 확인한 page를 기록 (Next()로 하나씩 읽는다 치면 어디서 멈췄는지가 필요함)
주로 다른 수단이 없을 때 쓰이는 용이지만, 생각보다 많은 최적화 존재
이미 앞 강의에서 많은 최적화를 다루었으며 뒤 강의에도 최적화 방법이 더 등장할 예정

### Data Skipping
Sequential Scan을 최적화하는 방법 중 하나

**Approximated Queries**
데이터의 부분집합을 적당히 샘플링해서, 근사적인 결과를 반환 -> Lossy하긴 함

**Zone Maps**
column별 aggregation을 미리 계산
-> 쿼리가 들어왔을 때 "이 페이지는 찾는 게 없으니 걸러라" 라고 말할 수 있게 됨

## Index Scan
만들어 놓은 것 중에 적당한 인덱스를 하나 고르고 scan
어떤 인덱스를 고를지는 나중 강의에서 자세하게 다룬다고 했지만, 직관적으로 data의 개수, predicate의 생김새 등을 보고 파악할 수도 있음

## Multi-index Scan
인덱스를 하나만 쓰라는 법은 없으니, 인덱스를 여러 개 써도 됨
각각의 index에 대해 lookup 해 보고, 합집합, 교집합 연산을 통해 원하는 결과를 얻어내기

# Modification Query

## Update/Delete
자식노드가 타겟이 되는 record의 ID를 부모노드에게 넘겨줌
이때 이미 확인한 노드가 어떤어떤 건지를 꼭 기록하고 있어야 함

### Halloween Problem
UPDATE 쿼리가 특정 'logical' tuple의 물리적 위치를 바꾸면, 같은 'logical' tuple을 여러 번 보는 상황이 생길 수도 있음
그러므로 이미 확인한 노드가 어떤 건지를 꼭 파악해야 하는 것임

## INSERT
operator 실행 도중 materialize하든지, 자식노드가 던진 거를 insert하든지

# Expression Evaluation
WHERE절 뒤에 나오는 수식으로 expression tree를 만들 수 있음
Expression Tree는 attribute, constant부터 시작해서 =, +, 등 연산자로 이루어짐
자구에서 중위표기식을 후위표기식으로 바꿀 때 트리 그린 거랑 비슷하게 생각하면 될 듯함
하지만 그냥 WHERE 뒤에 나오는 수식을 평가하느라 트리 하나를 탐색해야 하는 건 아무래도 느려 보임

## Optimizations
constant folding: 서브트리가 constant expression이면 하나로 합치기
common subexpression elimination: DP처럼, 같은 subexpression이 여러 번 쓰이면 한 번만 직접 구하고 다음부터 재사용

# JIT
아니면 WHERE 뒤에 있는 식을 평가하는 프로그램(C, Assembly같은 거)을 컴파일시켜서 그걸로 하는 방법도 있음 -> 이렇게 할 때 빨라지는 경우가 꽤 많이 존재
