---
title: 11 Sorting & Aggregation Algorithms
author: jaeyeol
layout: post
---

이제부터는 operator execution 파트로 넘어갈 예정

# Query Plan
relational algebra에서 배웠던 연산자들이 트리 구조로 되어 있음

# Sorting
기본적으로, relational model에서는 데이터가 정렬된 상태로 있지 않음
이때 SQL에서는 ORDER BY절을 이용해 데이터를 정렬된 상태로 불러올 수 있음
아니면, 이 정리 노트 후반부에 다루겠지만 쿼리가 정렬하라고 시키지 않아도 쿼리를 처리하기 위해 정렬하는 게 이득인 경우도 있음
만약 정렬할 데이터가 메모리 안에 전부 들어올 수 있다? 그냥 원하는 정렬 알고리즘 써서 정렬하면 끝.
물론 여기서, 데이터가 "거의" 정렬되어 있음을 알고 있다면 그것을 이용하는 VergeSort 등의 최적화가 가능하긴 함
아무튼 데이터가 메모리에 fit하지 못하는 경우를 상정
입력 단위를 run이라고 부르는 거 같다. why??
key: 정렬의 기준이 되는 attribute, 당연히 여러 개일 수도 있음
value: 두 가지 선택지 존재
row store 형식의 경우, early materialization 방법이 유용함. 이 방법은 key에 tuple 자체를 붙이는 방법
column store 형식의 경우, late materialization 방법이 유용함. 이 방법은 record ID나 오프셋 정도만 붙이는 방법

## Top-N Heap Sort
쿼리들 중에서 LIMIT 절 등 정렬 결과상 앞에서부터 몇 개의 데이터까지만 출력하라는 쿼리가 주어지는 경우가 있음
이 경우 그냥 전체 데이터를 한 번만 스캔해서 힙에 넣은 뒤 heap sort를 해 주면 쉬움

## External Merge Sort
그렇지 않은 general한 경우에는 divide and conquer 알고리즘인 external merge sort를 사용함
이 알고리즘의 경우 두 개의 phase로 이루어짐
phase 1: Sorting
이 단계에서는 chunk를 메모리에 불러서 정렬한 다음 디스크에 넣음
phase 2: Merging
정렬된 데이터 뭉탱이가 있다는 것을 알고 있으므로 합침
일반적인 merge sort와는 다르게 page 하나가 최소 단위인 듯?? 확실하진않음
아무튼 모든 데이터를 한 번 훑는 걸 pass라고 정의
한 번의 pass에서는 sequential I/O로 진행됨
정렬해야 하는 페이지 개수가 N이면, ceil(1 + logN)번의 pass가 필요하며 (pass 횟수) * 2N번의 I/O가 필요
하지만 이렇게 하면 Buffer Pool에 공간이 많아도 제대로 이용하지 못함
Pass 0에서는, buffer page B개를 사용해 크기가 B인 sorted run을 생성
그 다음 Pass부터는 B - 1 개의 run을 합침
따라서 총 필요한 pass 횟수는 1 + ceil(log_(B-1) ceil(N/B))가 됨

## Optimizations

### Double Buffering
Disk I/O 하는 데 시간이 매우 오래 걸리기 때문에, 지금의 방식으로는 CPU가 아무것도 안 하는 시간이 큼
그러므로 buffer 중에서 절반을 다음에 해야 될 것을 prefetch하는 데 사용

### Comparison Optimization
비교해야 할 대상이 문자열인 상황 등 key가 복잡하면 비교 연산 자체가 비싸짐
따라서 세 가지 정도의 최적화 방법을 떠올릴 수 있음
Code Specialization: C++의 template처럼, 해당 key를 정렬하는 정렬함수를 하드코딩 (compare함수로 넣지 말고)
Suffix Truncation: 앞부분 일부만 우선 가져와서 binary로 만들고 그걸 비교. (예시: 앞 몇 비트를 가져와 정수로 만들기)
Key Normalization: 값이 가변길이면 고정길이이면서 순서는 그대로인 형식으로 바꾸기

## Using B+Tree for Sorting
B+Tree가 있다면 당연히 쓰면 좋을 거 같음
만약 정렬해야 될 순서가 B+Tree가 정렬된 순서와 같다면 완전 이득
그냥 leaf node를 한 바퀴 쭉 돌면 정렬된 결과를 얻을 수 있게 됨
하지만 그렇지 않다면, N이 매우 작은 top-N 쿼리가 아닌 이상 random I/O를 유발하므로 매우 느림
이럴 바에는 차라리 external merge sort를 하는 게 더 나음

# Aggregations
주로 sorting, hashing 중 하나의 방법으로 aggregation이 구현됨

## Sorting Aggregations
데이터가 정렬되었다고 가정하면, aggregate는 데이터를 한 번 sequential scan하면 되므로 매우 쉬움
그렇기 때문에 aggregation을 하기 전에 우선 데이터를 정렬해 보는 것도 생각해 볼 만한 방법임
예시: 중복 제거같은 상황의 경우 ORDER BY를 직접적으로 쿼리에서 요구하기 않더라도 먼저 정렬하면 그 다음에는 sequential scan 한 번으로 충분함
하지만, ORDER BY같은 게 직접적으로 없다면 hashing 방법을 사용하는 게 일반적으로 낫다고 알려져 있음

## Hashing Aggregation
Idea: 메모리 안에 임시적으로 해시 테이블을 만들자
이때 해시 테이블을 메모리에 다 만들 만큼의 공간이 있다면 매우 쉬움
그렇지 않으면 External Hashing Aggregate 방법을 사용해야 함
이 방법도 DnC 방법이며 두 개의 phase로 나누어짐
phase 1: partition
일단 특정한 해시함수를 적용해서 key를 기준으로 튜플들을 각각의 버킷에 나누고, 버킷이 가득 찼다면 disk에다가 출력
phase 2: re-hash
위에서 적용한 것과 다른 해시함수를 써서, 메모리 안에 해시 테이블을 만들고 aggregate하기
모든 데이터를 다 넣을 해시 테이블은 만들지 못하지만 이미 partition된 데이터이므로 괜찮음
