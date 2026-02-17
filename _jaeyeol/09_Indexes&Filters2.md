---
title: 08 Indexes & Filters 2
author: jaeyeol
layout: post
---

Index: 데이터가 존재함? 존재하면 어디에 있음? 에 대한 대답  
Filter: 데이터가 존재함? 까지에 대해서만 대답  

# Bloom Filter
Set Membership을 테스트하기 위한 비결정적인 자료구조  
false negative는 존재하지 않음이 보장됨, 그러나 false positive가 존재할 가능성 있음  
insert, lookup 연산만을 지원. 즉, update나 delete는 지원하지 않음  
아이디어: bit map과 해시함수 여러 개 사용  
비트 여럿을 담은 배열을 관리, 처음에는 전부 0으로 초기화  
insert 연산: 각각의 해시함수 결과가 가리키는 인덱스의 비트를 1로 바꾸기  
lookup 연산: 모든 해시함수 결과가 가리키는 인덱스의 비트가 다 1이면 데이터가 존재한다고 판단  

## Counting Bloom Filter
비트 여러 개가 아니라 자연수 여러 개를 담은 배열을 관리  
특정 key가 몇 번 들어갔는지 확인할 수 있으므로 delete 등의 연산이 가능해짐  

## Cuckoo Filter
Cuckoo Hash Table을 사용하며, key값 전체 대신 'fingerprint'라는 일부분을 사용  

## Succinct Range Filter (SuRF)
Trie 구조에 속하며, 점쿼리 및 구간쿼리를 처리 가능  

# Skip Lists
B+Tree의 맨 아래 레벨은 정렬 상태임이 보장되는 링크드 리스트라고 할 수 있음  
dynamic하며 정렬상태를 보장해주는 인덱스를 구현할 때는 sorted 링크드리스트가 간단하고 쉬움  
하지만 최악의 경우 linear search를 하게 되면 값을 찾는 데에만 O(N)까지 걸리게 됨  
Skip List의 경우, 링크드 리스트를 여러 개의 계층 구조로 관리  
첫 계층은 B+Tree의 맨 아래 레벨처럼 온전하게 정렬된 링크드 리스트를 저장  
다른 계층들은, 대략 자기 바로 아래 계층이 가지고 있는 노드의 절반 정도만 가지고 올라오도록 함  
계층 번호를 아래에서 위로 갈수록 높게 매긴다고 가정  
- Insert - logical
1계층에서는 그냥 추가  
2계층부터는 넣고 있는 key를 이 계층에 넣을지 말지를 랜덤하게 결정
넣는 선택지가 골라졌다면, 해당 계층에 인서트하고 다음 계층으로 가서 넣을지 말지 판단
넣지 않는 선택지가 골라졌다면, 그 자리에서 종료
- Insert - physical
주로 메모리에 올라가는 자료구조이므로, 새로 만들어질 노드들이 들어갈 공간을 우선 만들어 놓고, 원래 있던 구조와 연결하지는 않음
그 다음 1계층에서 새로운 노드와 그 노드의 오른쪽을 이어줌  
그 다음 1계층에서 새로운 노드의 왼쪽과 새로운 노드를 이어줌
위 과정을 모든 계층에 반복한 뒤 위 계층에서 아래 계층으로 포인터를 통해 이어 줌
- lookup
항상 맨 위 계층에서 시작
각 계층에서는 링크드 리스트 순서상 다음 노드를 보고, 다음 노드의 key값이 lookup할 key값보다 작으면 그 다음 노드를 탐색, 아니면 내려감
- delete
멀티스레드 환경 대응을 위해, 그리고 physically 지우는 과정은 오버헤드가 크므로, logical하게 지운 후 physical하게 지우는 과정을 거침
logical하게 지우는 과정은, "이 노드는 무시해라" 라는 내용의 flag를 켜면 됨
그 이후 어떤 스레드도 지워질 노드를 참조하지 않게 되면 physical하게 지움. 이 과정은 insert의 역연산

장점: 만약 singly linked list 형태로 만들면 메모리를 적게 사용  
삽입과 삭제 과정에서 rebalancing이 필요없음

# Trie
B+Tree의 inner node에 어떤 key가 있다고 해도 그 key가 실제로 리프노드에도 존재한다는 보장이 없음  
즉, B+Tree에서 특정 key가 존재하는지 판단하려면 무조건 끝까지 내려가봐야 함  
Trie의 경우 각각의 노드에 key의 prefix를 몇 글자씩 저장  
key에 속한 모든 글자를 다 돌고 나면 그제서야 key에 해당하는 포인터를 저장  
이렇게 하면 key 문자열의 길이가 k일 때 O(k)만에 찾을 수 있게 됨
span: 각각의 노드에 prefix를 얼마나 저장할지 결정
이게 결정되면 각 노드의 최대 자식 수와 트리의 높이 등이 결정됨
예시로 key가 {0,1}*에 속할 때 Trie를 만들었다고 가정하면, 각 노드는 (0, ptr0, 1, ptr1)의 형태가 됨  
이때 0으로 시작하는 key가 있다면 ptr0은 자식노드의 포인터, 아니면 null이 됨. ptr1도 마찬가지  
당연하게도 (0, ptr0, 1, ptr1)보다는 (ptr0, ptr1)만 저장하는 게 효율적
또한 특정 노드의 자식 노드가 하나밖에 없는 경우에도 비효율적이 됨  
이를 해결하기 위해 자식 노드가 하나밖에 없는 놈들을 압축해버린 것을 Radix Tree라고 부름
물론 Radix Tree로 넘어오는 순간 false positive가 생길 수 있음  
key값으로 1100011, 0000000이 들어와 있는 상황에서 1111111을 읽으면, 중간에 있는 000은 압축되어 사라졌을 것이므로 1111111이 존재한다고 잘못 판단하게 됨

변형 자료구조로 Judy Array, ART Index, Masstree, SuRF 등이 존재

# Inverted Index
지금까지 본 방법들은 전부 점쿼리나 구간쿼리에서 뛰어난 성능을 보여줌  
하지만 SQL의 LIKE 구문과 같은 keyword search는 제대로 수행하지 못함  
이 문제를 해결하기 위해 Inverted Index는 긴 텍스트가 주어지면 텍스트를 구성하는 각각의 단어를 꺼내와서 그걸로 dictionary 자료구조를 만듦
dictionary는 특정 단어를 그 단어가 들어가는 row들로 매핑시켜줌
Case Study(Lucene): 'finite state transducer'이라는 그래프를 만들어 특정 단어가 들어왔을 때 이를 딕셔너리의 어느 offset으로 보낼지를 효율적으로 정함
transducer은 한 번 만들면 바꿀 수 없기 때문에 새로운 값이 들어오면 incremental하게 새로 만들고 merge해야 함
Case Study(Postgres): Generalized Inverted Index(GIN) 사용, 이는 딕셔너리가 B+Tree, 리프노드는 'posting list'라는 자료구조를 사용
posting list는 안에 있는 단어 수가 적으면 array, 많으면 다른 B+Tree 사용
이 posting list를 관리하는 오버헤드가 생기므로 pending list라는 다른 자료구조에 mod log 저장
Inverted Index의 응용 예시로는 ranking, tokenizer 등이 있음

# Vector Indexes
Inverted Index의 경우에도 단어가 정확히 일치하는 경우에만 찾을 수 있음  
즉 특정 단어와 의미적으로 가까운 단어가 들어간 레코드를 찾는 등의 일은 수행하지 못함  
semantic similarity search의 경우 DB에 들어간 텍스트를 transformer을 이용해 임베딩 벡터로 바꾸고, 이것을 벡터 인덱스에 집어넣음  
쿼리가 들어왔을 때도 마찬가지로 임베딩 벡터로 바꾼 뒤 벡터 인덱스의 값들과 nearest neighbor search를 수행
그래서 vector index가 뭐냐??
임베딩 벡터들끼리 nearest neighbor search를 잘 하기 위해 만들어진 놈으로 두 가지 방법 정도가 있음

## Inverted Indexes
전체 벡터를 클러스터링 알고리즘으로 여러 뭉탱이로 쪼갠 다음, 각 클러스터의 중심 -> (해당 클러스터에 속한 모든 record)로 매핑해주는 inverted index를 만듦
lookup할 때는 같은 클러스터링 알고리즘으로 어느 클러스터에 속해있는지 확인해주면 됨

## Graph Search
각각의 노드를 각각의 벡터에 대응시키는 그래프를 하나 만들고, 모든 노드는 최대 n개의 nearest neighbor하고만 연결되어 있도록 함
특정한 벡터가 쿼리 입력으로 주어졌을 때, 그래프의 아무 정점에서나 시작해 쿼리 벡터와의 거리를 좁히는 방향으로 그리디하게 탐색
노드 개수가 너무 많아지면 빠른 시간 내에 처리할 수 없으므로 HNSW 등의 최적화가 존재
HNSW는 graph search에 skip list스러운 아이디어를 합쳤다고 볼 수 있음  
맨 아래 레벨은 온전한 그래프, 그 위 레벨은 랜덤하게 일부만 남김

# Partial Index
CREATE INDEX문에 WHERE조건을 추가할 수 있음
인덱스를 유지하는 오버헤드가 줄어들며, SELECT 등에서 WHERE 조건에 인덱스를 만들면서 설정했던 조건이 같이 있으면 편해짐
예를 들어, 한 달에 한 번씩 partial index를 만들어두면 인덱스를 날짜 범위 기준으로 나눌 수 있게 됨

# Index Include Columns
CREATE INDEX문에 INCLUDE(column) 추가
예를 들어 key가 (a, b)이고 INCLUDE(c)를 추가했다고 하자
그러면 B+Tree 리프노드에 a, b와 함께 c도 같이 저장
당연하지만, 그렇다고 해도 c값을 key로 쓰지는 않음
만약 쿼리가 원하는 모든 attribute가 B+Tree 안에 다 있으면 굳이 튜플 전체를 가져올 필요가 없게 됨
