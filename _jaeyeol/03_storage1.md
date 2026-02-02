---
title: 03 Database Storage I
author: jaeyeol
layout: post
---

이제부터 어떻게 소프트웨어를 build할지 알아볼 예정  

DB System: 레이어 뭉탱이  
Query Planning  
Operator Execution  
Access Methods  
Buffer Pool Manager  
Disk Manager  
이 중 3회차에서는 Disk Manager에 대해 알아볼 예정  

# Disk-Based Architecture
Memory Hierarchy를 복습해보자.  
CPU Register  
CPU Cache  
DRAM  
SSD  
HDD  
Network Storage  
위로 갈수록 비싸고 빠르고 작음, 아래로 갈수록 싸고 느리고 큼  
이 중 CPU Register, CPU Cache를 묶어서 CPU, DRAM을 Memory, SSD, HDD, Network Storage를 묶어서 Disk라고 부름  
또한 CPU Register, CPU Cache, DRAM의 경우 Volatile함 (Random Access 가능, Byte_Addressable)  
반면 SSD, HDD, Network Storage의 경우 Non-Volatile함 (Sequential Access만 가능, Block_Addressable (뭉탱이 단위로만 접근할 수 있음))  
Sequential I/O와 Random I/O에는 속도에 있어 큰 차이가 있으므로 되도록 Random I/O의 사용을 피함  
DBMS 또한 Sequential Access의 사용을 최대화해야 함  

# Problem
두 가지 문제를 살펴볼 예정  
1. DBMS가 Disk File 안에서 DB를 어떻게 표현할 것인가?
2. DBMS가 어떻게 메모리를 관리하고 메모리/disk 사이 데이터를 이동시킬 것인가?
3회차에는 1번 문제, 4회차에는 2번 문제에 대한 답을 줄 예정

# DB 표현
Storage Manager: DB 파일 관리, 페이지 뭉탱이로 관리  
physical level에서는 같은 페이지를 여러 개 만들지 않음 (클컴때 배운 같은거 복사한다거나 그런거는 physical level에서는 원래는 안 일어나는 일임)  
**Page**: 데이터 블록인데 크기가 고정임, 페이지마다 고유 ID 존재  
크기는 여러 요인에 의해 결정
예시: read_heavy이면 크게, write_heavy이면 작게  
페이지 관리: Heap, Tree, ISAM, Hashing 등등...  
그중 Heap File: 순서 없는 페이지들의 뭉탱이  
Page Directory: DB 안의 DB라고 볼 수 있음, 각 page의 위치, 오프셋을 기록, sync가 제대로 맞는지 확인

# Page Layout
헤더: 메타데이터 포함  
데이터: Tuple 등등, 여기서는 row-oriented model을 가정  
Tuple Size끼리 같으면 array 접근하는 것처럼 할 수 있음  
데이터가 지워져서 빈 게 생기면 가장 앞에 있는 빈 슬롯에 넣기... 시간오래걸림 + compaction 힘듦

# Slotted Pages
많이 쓰는 솔루션  
처음에 헤더와 slot array 존재, 이는 각 튜플의 오프셋을 저장하는 배열임  
장점: 튜플 지워도 slot array만 고치면 됨, memory에 올렸을 때 수정 cost도 적음  
logical level의 튜플을 physical level의 위치에 매핑하는법?  
Record ID를 사용할 수 있음... 그러나 요즘에는 런타임 도중에 값을 유도함  
SQLite의 경우 ROWID를 숨겨진 PK로 사용 but 값이 바뀌므로 믿을 수 없음

# Tuple Layout
Core: 바이트의 array + header, 이때 헤더는 페이지의 헤더에 들어있는 데이터와 안겹치게  
여러 word에 걸치는 경우 등에는, byte alligned 되도록 zero padding을 추가하든지, 더 잘 allign 되도록 값 재정렬  

# Data Representation
INT 계열: C/C++과 동일  
FLOAT/REAL: IEEE 754  
NUMERIC/DECIMAL: fixed point representaiton  
VARCHAR 등 문자열 계열: 헤더에 길이 저장 + data byte 또는 pointer  
Timestamp 계열: Unix의 time since epoch인 경우가 많음  
fixed point representation의 경우, 정밀도를 마음대로 조절할 수 있지만 DBMS 차원에서 구현되어 CPU 연산 한번에서 끝나지 않으므로 오래 걸림  
NULL의 경우 보통은 헤더에 어느 column이 NULL인지 bitmap 저장  
매우 용량이 큰 attribute의 경우 별도의 페이지 사용, 원래 tuple에는 새 페이지로 가는 포인터와 데이터 크기만 저장  
이를 최적화하기 위해 페이지 압축, German String등의 방법 사용  
어떤 시스템은 아예 외부 저장소를 사용하기도 함: BLOB 타입으로 저장, URI나 포인터 저장  
