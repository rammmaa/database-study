---
title: 08 Indexes & Filters 1
author: jaeyeol
layout: post
---

이번 8강의 주 목표는 B+Tree에 대해 알아보는 것

# B+Tree
Balanced, Ordered 성질을 가지고 있고 각 노드에 m개의 포인터가 달려있음
m이라는 값은 B+Tree를 만들 때 사람이 정하는 값인 듯
몇 가지 중요한 성질
1. 모든 리프노드의 깊이가 동일
2. 루트노드를 빼면 모든 노드가 m/2 - 1개 이상의 자식노드를 가짐
3. inner node는 어떤 k에 대해 k개의 키와 k+1개의 null이 아닌 자식노드를 가짐
4. 큰 데이터를 읽고 쓸 때에 대비해 최적화되어 있음

보통 m을 크게, 높이를 작게 (4나 5 정도) 로 함
루트가 아닌 노드는 같은 깊이의 바로 옆 노드를 가리키는 Sibling Pointer을 추가로 들고 있음
의의: 그냥 링크드 리스트만 있었다면 특정 키 찾는 데 오래 걸렸을 텐데 이분탐색 비슷하게 갈 수 있어서 빨라짐

## Node
기본적으로 key, value 쌍들을 모아 놓은 array라고 할 수 있음
개념적으로는 리프노드의 경우 (빈칸) key val key val 형식, 리프노드가 아닌 경우 node포인터 key node포인터 key node포인터 형식
실제 시스템에서는 깊이, 비어 있는 슬롯 개수, 이전 노드로 가는 포인터, 다음 노드로 가는 포인터 등을 메타데이터로 저장
그 다음, key value 쌍을 저장
value에 들어갈 값을 명확히 하지 않았는데 두 가지 방법이 있음
1. Record IDs
   가장 흔하게 쓰임
2. Tuple Date
   Index-oriented Storage에서 봤었던 거 같은 모양
   - Primary Key Index: 실제 데이터를 담음
   - Secondary Index: PK에 해당하는 값만 담음

# B Tree vs B+Tree
B Tree는 inner node 어디든 value를 저장할 수 있음
반면에 B+Tree는 value를 리프노드에만 저장
그에 따라 공간적으로 효율적인 건 B Tree 쪽임
하지만 B Tree의 경우 원소를 순회한다거나 하면 DFS가 필요하고 Random I/O가 필요하므로 B+Tree가 시간적으로 훨씬 효율적임
심지어 삭제 연산도 B+Tree가 더 간단함 (B+Tree에서는 키를 하나 지워도 지워진 키가 inner node에 남아있는 건 상관없음)

## Insertion
삽입할 데이터가 어떤 리프 노드에 속하는 범위인지 우선 확인
그 리프노드를 L이라고 하면, L 안의 key가 정렬되었다는 사실을 깨지 않으면서 삽입
이때 오버플로우가 되지 않으면 행복한 거고, 오버플로우가 나버리면 노드를 쪼개야 함
쪼개는 과정은 다음과 같음
L 바로 옆에다가 L2라는 새로운 노드를 만들고
L에 있던 값과 새로운 값을 L, L2에 고르게 넣음
그리고 중앙값 (즉 L2의 처음에 오게 된 값)을 복사해서 부모노드로 올림
부모노드에서도 그에 따라 L2로 가는 포인터를 추가
만약 부모노드에 값을 추가하려 할 때 그것마저 오버플로우이면 위 과정을 반복
다만 리프노드가 아닌 경우 중앙값을 복사해서 부모노드로 올리는 게 아니라 그냥 올림

## Deletion
Insertion과 같은 방법으로, 리프 노드 L을 찾고 삭제
삭제하고 나서도 half_full 이면, 즉 지운 값이 속해 있던 노드에 m/2 - 1 개 이상의 값이 있다면 행복
그렇지 않으면 이번엔 노드 두 개를 합쳐야 함
바로 왼쪽에 있는 노드나 오른쪽에 있는 노드에 가서, 해당 노드에서 값을 뺏어와도 그 노드가 안전하다면 값을 뻇어옴
그렇지 않으면 L과 sibling 노드를 합치고, insertion 처리할 때처럼 부모노드로 올림

# Composite Index
Key에 꼭 어트리뷰트 하나만 들어가라는 법은 없으니, 어트리뷰트 여러 개가 들어간 경우를 composite index라고 부름
이 경우에도, 만약 쿼리가 composite key의 prefix만 묻는 경우 (예를 들어 키가 (a, b, c)인데 쿼리가 (a = 2 AND b = 4)를 묻는 경우) 여전히 B+Tree를 이용해 단순하게 처리 가능
그렇지 않은 경우 (예를 들어 키가 (a, b)인데 쿼리가 (b = 1)인 경우)에도 모든 노드를 다 볼 필요까지는 없을 수 있음
어차피 a가 같은 노드들 사이에서는 b끼리 정렬된 상태이므로 그 부분만 확인해서 parallel하게 돌리면 됨

# Duplicate Keys
실제 상황에서는 같은 데이터의 여러 버전을 관리한다든가 그런 이유로 키가 중복될 가능성이 존재
이를 해결하기 위해 두 가지 정도의 방법을 제시
1. Append Record ID
   숨겨진 어트리뷰트인 record ID를 만들어 같이 저장, 이 record ID는 고유하도록
2. Overflow Leaf Nodes
   진짜 그냥 리프노드에 다 때려넣는 방식인데, 쓰지 않음

# Clustered Indexes
Table이 PK 기준으로 정렬된 것을 말함
Table Page가 정렬된 순서로 되어 있을 때 리프노드를 sequential scan하면, 그에 따라 table page 또한 sequential scan하게 되어 빠름
이 성질을 만족하지 못할 경우, 리프노드를 sequential scan할 때 사실상 page를 random access하게 됨
이건 싫으니까, 스캔 도중에는 각 튜플의 페이지 번호만 미리 가져오고 페이지 자체를 가져오지 않음
그리고 스캔이 끝나면 그걸 다 정렬한 다음에 보면 됨

# Design Choices
B+Tree를 설계할 때 생각해야 할 여러 가지 요소

## Node Size
최적 사이즈는 하드웨어에 따라, workload에 따라 달라짐
일반적으로 하드웨어가 느릴수록 큰 사이즈가 최적임 (SSD에서는 더 크게, 메모리에서는 더 작게)

## Merge Threshold
노드를 지운 뒤 합치는 연산은 비싸니, 지운 뒤 반만 남았다고 바로 합치지 않기
물론 일시적으로 B+Tree의 성질이 깨지긴 하지만, 합치는 연산을 미루니 Disk I/O를 줄여서 빨라짐

## Variable-length Keys
1. Pointers
   키를 attribute의 포인터로 담기
   이렇게 하는 걸 T Tree 라고 함
2. Variable-length Nodes
   예상되듯 메모리 관리가 어려울 수 있음
3. Padding
   가장 긴 놈에 맞춰서 나머지 것들에 padding을 채움
4. Key Map / Indirection
   Slotted Page랑 비슷하게 앞에서부터 offset들을 저장하고 뒤에서부터 데이터 저장

## Intra-node Search
노드 안에도 키가 여러 개 있으니 그것들 중 원하는 걸 어떻게 빠르게 찾을지도 생각해봐야 함
1. Linear
   진짜 앞에서부터 찾기 (SIMD같은 걸로 벡터화 가능)
2. Binary
   진짜 이분탐색
3. Interpolation
   데이터가 조밀하게 있을 때만 사용가능, 근사적으로 위치를 먼저 구함

# Optimization
## Pointer Swizzling
원래는 한 노드에서 다른 노드로 가는 포인터에 Page ID를 사용함
그러나 Buffer Pool에 노드가 고정되어 있을 경우 Page ID 대신 진짜 포인터를 대신 저장해도 됨
이러면 hash table 등에 접근할 필요가 없으므로 빠르지만, 그렇게 저장해놓고 노드가 없어지면 포인터가 사라지므로 그러지 않게 주의 필요

## Write-Optimized B+Tree
Fractal Tree, B-epsilon Tree 라고도 불림
동작방식은 lazy segment tree와 유사함
값의 추가, 삭제가 일어나면 inner node에 mod log를 추가 (처음에는 루트 노드에)
그러다가 특정한 노드의 mod log가 가득 차면 그걸 자식으로 전파
쿼리가 들어오면 먼저 아래로 내려가기보다 mod log를 확인
