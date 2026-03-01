---
title: 10 Index Concurrency Control
author: jaeyeol
layout: post
---

8, 9회차에서 다루었던 자료구조는 모두 single-thread 환경을 가정했었다
하지만 현대 DBMS의 경우 여러 CPU 코어를 이용하고 disk I/O로 인한 병목을 다른거 같이 하면서 숨기고 하느라, 한 자료구조에 여러 스레드가 접근할 수 있도록 해 주어야 함

# Concurrency Control
concurrency control protocol: 여러 스레드가 특정 오브젝트에 병렬적으로 연산을 날려도 올바른 결과를 보장해주는 방법
`올바른` 에는 logical과 physical 레벨이 있으며 이번 회차에서는 특별히 physical에 초점

# Locks vs. Latches
Locks: Transaction 단위, high-level, 롤백 기능이 필요함
Latches: 연산 단위, low-level, 롤백 기능이 필요하지 않음

# Latch Modes
Read Mode: 어차피 읽기만 하므로, Read 요청만 있는 경우 여러 스레드가 같은 데이터가 동시에 접근해도 됨
Write Mode: 한 스레드만 접근 가능해야 함, Write 끼리는 물론이고 Read와 Write끼리도 imcompatible

# Implementation Goals
- Latch 동작 중 사용하는 메모리가 적어야 함
- 다른 latch가 없을 때 execution path가 빨라야함(??)
- 뭔가 중앙해서 전부 관리해주는 그런 시스템이 없어야 함 (있다면 그런 걸 관리해주는 것 자체가 비싸서 오래 걸림)
- OS가 끼어드는 걸 최소한으로 줄여야 함

# Implementations
## Test-and-Set Spin Latch (TAS)
특징: Latch를 켜고 끄는 게 매우 효율적, 그러나 scalable하지 않고 cache-friendly하지 않고 OS-friendly하지 않음
예시: `std::atomic<bool>`
만약 특정 데이터에 접근해야 하는데 그 데이터의 latch를 다른 스레드가 가지고 있다면, 내가 latch를 가져갈 수 있을 때까지 계속 접근 시도
다만 이 과정에서 매 반복마다 데이터가 있는 곳으로 가서 latch가 있는지 없는지를 봐야 하므로 비쌈

## Blocking OS Mutex
특징: 간단함, 그러나 non-scalable
예시: `std::mutex`
`std::mutex`에는 `.lock()`, `.unlock()` 메소드가 있어, 다른 스레드가 접근하지 말아야 하는 부분에서 `lock`을 해주면 됨
하지만 이 경우, 특정한 데이터에 read 요청만 매우 많다고 하면 그냥 그걸 여러 스레드가 한번에 읽게 해 주면 빠르지만 이 경우 그걸 할 수 없음

## Reader-Writer Latches
특징: reader 스레드가 여럿이어도 됨, 그러나 read/write 스레드들의 queue를 관리해야 함
예시: `std::shared_mutex`

# Hash Table Latching
Hash Table의 경우, 데이터를 여러 개 조회해야 해도 모든 스레드가 다 같은 방향으로만 이동함
또한 모든 스레드가 한 번에 하나의 page/slot만 방문함
그렇기 때문에, hash table상의 경우 deadlock을 걱정할 필요가 없음
latch를 어디에다가 걸어 줄지에 따라 방법이 나뉨

## Global Latch
Latch 단 하나만을 이용해 그 latch가 자료구조 전체를 보호하게 됨
resize가 일어나고 있는 경우 무조건 global write latch 필요

## Page/Block Latch
Page/Block 하나마다 각각의 latch가 존재
한 스레드가 데이터를 조회할 경우, latch 얻어오기 -> page/block에 접근 (r/w) -> latch 놓아주기 -> 다음 데이터로 이동 순서대로 가야 함

## Slot Latch
Slot 하나당 각각의 latch 사용
이때 r/w latch를 나누지 않고 single mode로 사용하면 메타데이터나 오버헤드를 줄일 수 있음

# B+Tree Concurrency Control
## Concepts
B+Tree의 경우 다음 두 가지 문제를 우선 생각해볼 수 있음
- 여러 스레드들이 특정 노드의 데이터를 동시에 건드리려고 할 경우
- 어느 스레드가 트리를 split/merge하는 동안 다른 스레드가 트리를 탐색하고 있을 경우

해결책은 Latch Coupling 또는 Latch Crabbing 이라고 불리는 방법임
기본적으로, 부모노드에서 자식노드로 이동할 때 우선 자식노드의 latch를 얻어오고 부모노드가 safe인 경우에만 부모노드의 latch를 놓아 줌
여기서 safe란, 해당 노드 아래에서 업데이트가 일어난다고 해도 split/merge되지 않는 노드를 뜻함
즉 insertion의 경우 꽉 차 있지 않은 노드, deletion의 경우 절반보다 많이 차 있는 노드

Read 요청의 경우, root의 read latch를 얻고, 자식노드의 read latch 얻기 -> 부모노드의 read latch 놓아주기를 리프에 도착할 때까지 반복
Write 요청의 경우, write latch를 다 가져가면서 내려가고, 중간에 safe 노드를 발견하면 그 조상 노드의 latch를 전부 놓아 줌
이때, 뭘 하든간에 일단 루트노드에 latch를 걸어 두면서 시작한다는 것을 관찰할 수 있고, 이게 실제로 병목현상을 일으킴

## Optimistic Latching Algorithm
해결책으로, 어차피 split/merge를 해야 하는 상황이 자주 안 나온다는 것을 이용
우선 write 요청이 들어오더라도, 안나오겠지~ 하고 루트부터 내려가는 도중 write latch 말고 read latch를 챙겨가면서 내려감
그랬을 때 실제로 split/merge가 필요 없는 상황이었으면 행복한 거고
그렇지 않으면 어쩔 수 없이 정석적인 방법으로 다시 실행

## Leaf Node Scan
Hash Table에서 그랬던 것처럼, B+Tree 상에서도 위에서 아래로만 간다면 deadlock을 걱정할 필요가 없음
그러나, 특정 구간에 속한 값을 조회하는 등 리프노드의 sibling pointer를 타고 다른 리프로 가야 하는 상황이 생긴다면 걱정해야 함
다시 말해, 리프 노드를 훑는 상황 한정, sibling pointer를 타고 갔는데 도착한 노드를 이미 다른 스레드가 쓰고 있는 경우 deadlock일 가능성이 있음
이 경우 나중에 온 스레드를 죽이고 처음부터 다시 시작
이게 어쩔 수 없는 게, lock과는 달리 latch의 경우 deadlock 탐지같은 기능이 없으므로 구현 디테일으로 정해 줘야 함

## B+Tree Write Operations
만약 위와 같은 상황이 write 요청 중에 일어났다고 하면??
스레드를 그냥 죽이면 안 되고 해당 스레드가 했었던 수정들을 전부 다 롤백하고 죽여야 함
그렇기 때문에 스레드 내부에 있는 저장 공간에다가 변경사항을 전부 기록 (write-set)
또한, split/merge를 롤백하는 것은 코스트가 크기 때문에 8회차에서 언급된 대로 잠시 B+Tree 조건이 깨진 상태로 놔 뒀다가 나중에 rebalance하는 최적화도 가능
