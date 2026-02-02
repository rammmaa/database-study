---
title: 04 Memory Management & Buffer Pool
author: jaeyeol
layout: post
---

# Problem
두 가지 문제를 살펴볼 예정  
1. DBMS가 Disk File 안에서 DB를 어떻게 표현할 것인가?
2. DBMS가 어떻게 메모리를 관리하고 메모리/disk 사이 데이터를 이동시킬 것인가?
3회차에는 1번 문제, 4회차에는 2번 문제에 대한 답을 줄 예정

# Spatial, Temporal Control
Spatial Control: 어디에 써야함??  
목표: 자주 같이 쓰이는 페이지를 물리적으로도 가까이에 저장  
Temporal Control: 언제 읽고 써야 함??
목표: disk에서 읽는 행위를 최소화  
또한 보통 시스템이 시작될 때 뭉탱이로 malloc하고 더 이상 malloc하지 않음 (추가로 malloc했을때 안되면...)  
그러므로 정보가 buffer pool 안에서 끝나는 게 바람직함  

Memory Region: 고정 크기 페이지들의 배열  
배열 각각의 원소는 **Frame**이라고 부름  
DBMS가 요청 시, disk에 있던 페이지를 복사해 frame에 넣음  
**Page Table**: 페이지 ID를 포인터에 매핑. Page Directory와는 완전 다른 개념  

뭔가 OS를 사용하지 말라는 얘기를 자꾸 함  
mmap? 은 자세히 모르겠지만 Physical Memory가 다 찼을 때 page fault가 일어나면 문제가 생기는 듯.  
직관적으로 OS는 SQL Query, Transaction 등에 대해 아무것도 모르므로 중요하게 쓰이는 정보를 evict해버릴 수도 있음  
Problem 1. Transaction Safety (OS가 수정된 페이지를 아무때나 flush 가능)  
Problem 2. I/O Stalls (DBMS가 메모리에 뭐가 있는지 모르게 되므로 stall됨)  
Problem 3. Error Handling (메모리 건드는 도중 evict되면, mmap을 안썼을때는 그 부분만 보면 되는데 쓰면 힘들어짐)  
Problem 4. Performance Issues  
결국 나와야만 하는 질문: 메모리가 꽉 찼을 때 뭘 버려야함??  
목표: Correctness, Accuracy, Speed, Metadata Overload

# LRU (1965)
시간 순서대로 이어진 링크드 리스트를 기록, 가장 오랫동안 안 쓰인 페이지를 evict  

# CLOCK (1969)
LRU를 근사적으로나마 할 수 있음 (그만큼 더 빠름)  
시간순서 대신 access bit를 저장  
clock이 돌아가면서 이전에 확인했을 때부터 지금까지 값이 쓰였는지를 확인, 안 쓰였으면 evict  

다만 이것들 둘 다 sequential scan을 할 때 최신성 정보가 사라지는, sequential flooding에 취약

# LFU (1971)
아이디어: 카운터를 하나 만들어서 카운터가 가장 작은 놈을 evict  
하지만 카운터를 관리하는 데 상수가 아니라 로그 시간이 드는 문제가 있음  
또한 갑자기 한 명이 분탕으로 10억번정도 방문하고 다시는 안 방문한 데이터의 경우 카운터가 쓸데없이 높이 잡힘

# LRU_K (1993)
아이디어: LRU 리스트 여러 개를 관리  
evict할 때 현재 시점과 가장 오래된 access time을 비교, 이 차이가 가장 큰 놈을 evict  
리스트가 여러 개이므로 한 번 방문하고 마는 정보를 덜 중요하게 생각할 수 있음  
추가로 최근에 evict된 데이터를 잠시 저장해 놓는 ghost cache 추가

# MySQL's Approximate
MySQL은 LRU_K를 근사하기 위해 아래의 방법 사용  
LRU처럼 링크드 리스트를 만들되 young, old로 구분  
처음 쓰인 페이지는 old에 추가, old에 존재하는 페이지를 또 쓰면 young에 추가

# ARC (2003)
LRU, LFU의 장점을 모두 가져가기 위해 리스트를 2개 관리, 데이터가 어떻게 들어오느냐에 따라 두 리스트의 크기 조절  

# 기타 방법론
쿼리에서 자주 쓰이지 않는 놈은 global buffer pool 공간이 아까우니 따로 만들어둔 side cache 사용  
Buffer Pool에 페이지를 넣을 때 중요한 놈인지에 대한 힌트 전달  

# Dirty Page 관리
dirty가 아니면 그냥 없애버려도 됨  
dirty인 경우 그 페이지가 먹고 있는 프레임을 다시 쓰려면 disk에 그걸 다 쓸 때까지 기다려야 함  
Background Writing: 주기적으로 dirty page를 disk에 작성

