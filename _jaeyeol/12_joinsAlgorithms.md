---
title: 12 Joins Algorithms
author: jaeyeol
layout: post
---

# Join은 왜 필요할까?
Normal Form으로 relation들을 쪼개고 나서 그걸 합칠 때 join이 필요함
우리의 포커스: inner equijoin (=INNER JOIN이며 ON 뒤에 A.val = B.val 등 =조건이 붙은 것)
3개 이상의 테이블을 한꺼번에 join하는 경우는 신경쓰지 않음

# Query Plan
Query plan상에서 join??
저번 시간에 살펴보았듯, inner join 연산도 query plan상에서 한 가지 과정이므로, 위로 결과를 뱉을 수 있음

## Operator Output
Early Materialization: input 값을 그대로 복사해서 output에 넣기
Late Materialization: joins keys, record ID 정도만 넘겨주기

## Cost Analysis Criteria
disk I/O가 파멸적으로 느리기 때문에, disk I/O를 몇 번 하는지만 생각할 예정

# Nested Loop Join
Notation: R inner join S 연산을 수행하야 하면, R을 outer table, S를 inner table이라고 부름
R은 M개의 페이지, m개의 튜플을 가지고 있으며, S는 N개의 페이지, n개의 튜플을 가지고 있다고 가정

## Naive Nested Loop Join
for r in R: for s in S: 형태로 반복
R의 각 원소에 대해 S를 한 번씩 스캔하므로 매우 비쌈
총 비용: M + mN

## Block Nested Loop Join
R, S를 여러 뭉탱이로 쪼갬
한 페이지를 한 블록으로 묶는다고 가정하면, M + MN
페이지를 불러오면 in-memory join을 할 수 있기 때문에 쉬움

만약 여기에 더해서 B개의 buffer 있으면 하나를 inner table에, 하나를 output에, 나머지 모두를 outer table에 부여하면
M + Nceil(M / (B-2))까지 줄어듦

## Index Nested Loop Join
nested loop join 계열이 느린 이유는, outer 값 하나마다 inner를 sequential scan해야 하기 떄문
이를 피하기 위해 잠깐 쓰고 버릴 index를 하나 만들어 주기
어떤 index에 probe하는 데 C만큼 걸린다고 하면, 총 I/O횟수는 M + mC

어느 경우든, 더 작은 테이블을 outer table으로, 더 큰 테이블을 inner table으로 두는 편이 시간 측면에서 더 이득임

# Sort-Merge Join
일반적인 상황에서 hash join보다는 느리지만, 데이터가 이미 정렬되어 있거나 ORDER BY 등이 있다면 이게 더 나음
sort-merge 라는 이름답게, 정렬한 뒤 합치는 두 가지 step으로 이루어짐
정렬: 11강에서 다룬 알고리즘을 포함한 원하는 아무 알고리즘이나 사용해서 양쪽 테이블 모두를 정렬
합치기: 투 포인터 방식, 중복값이 있을 경우에 처리가 좀 귀찮긴 함
일반적인 경우, 비용은 (정렬 비용) + M + N
하지만, 모든 값이 다 같은 최악의 경우 비용은 (정렬 비용) + MN까지 늘어남

# Hash Join
해시함수를 이용해 테이블을 적당히 partition들로 나눈다면, 같은 partition 안에 있는 값들만 확인해도 됨

## Simple Hash Join
Phase 1: Build
outer table의 값들에 대해 hash table을 만들어줌
Phase 2: Probe
inner table의 값들을 해시 함수에 집어넣고 hash table에 있는 값들과 비교
index nested loop join과 메커니즘 자체는 비슷하지만 인덱스 대신 해시함수를 사용했다고 생각하면 대충 됨
Probe Filter: bloom filter같은 걸 만들어서 걔가 긍정 답을 뱉으면 그때 해시 테이블에 넣기

## Partitioned Hash Join
위 알고리즘은 어쨌거나 메모리 안에 전체 해시 테이블을 다 만들 만큼의 공간이 있다는 것을 가정함
그렇지 않은 경우를 처리하기 위해 동작 과정을 수정
Phase 1: Partition
R, S 양쪽을 다 해시해서 bucket 단위로 쪼갬
Phase 2: Probe
각각의 bucket끼리 simple hash join

만약 여기서 각 partition마저도 메모리 안에 들어가지 못하면 어떡할까?
답: 다른 해시함수를 이용해, 메모리에 들어갈 때까지 recursive partition

만약 같은 key가 너무 많아서 애초에 저렇게 해결할 수 없다면?
답: 해당 key에 대해서만 block nested loop join을 수행

비용은 recursive partition이 필요없을 경우 3(M+N)

### Hybrid Hash Join
만약 key의 분포가 고르지 않음을 알고 있다면 몰린 쪽을 in-memory join하고 나머지를 일단 disk에 보내버릴 수 있음



