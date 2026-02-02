---
title: 05 Database Storage II
author: jaeyeol
layout: post
---

# Buffer Pool 최적화
- 여러개 만들기
  -  Record ID 기준으로 Buffer Pool ID 매핑
  -  Hashing
- Prefetching
  DBMS는 쿼리 수행 계획에 따라 나중에 보게 될 page를 미리 buffer pool로 불러올 수 있음
- Scan Sharing
  여러 쿼리가 같은 커서에 붙을 수 있도록 허용 (같은 테이블에 sum, max 쿼리가 들어왔다 치면 테이블을 2번 대신 1번만 스캔해도 됨)

# Storage
- Tuple-Oriented Storage
  - Read: Record ID 가져오고, page directory에서 page 위치 가져오고, page에서 데이터 읽음
  각각의 storage 찾을 때 `index` 사용
  - Insert: free slot 있는 거 찾음
  - Update: 데이터가 들어있는 page를 찾고 덮어쓰기... 그러나 데이터가 너무 크면 다 지우고 새 페이지에 써야 함...
- Index-Organized Storage
  이번에는 DBMS가 tuple을 정렬된 상태로 유지
  트리 구조처럼 만들고, 리프 노드에 Record ID 저장
  - Read: Inner Node 스캔해서 리프 노드로 내려오고, key/offset을 이분탐색으로 찾음
- Log-Structured Storage
  이미 page에 저장된 데이터를 overwrite할 수 없고 append만 가능하다고 가정
  - Idea: in-place로 값을 바꾸는 대신, 변경사항을 기록하는 로그를 저장 (PUT, DELETE)
  - 이제 변경사항을 메모리 안의 MemTable이라는 자료구조에 저장, disk에다가는 SSTable이라는 자료구조에 sequentially 넣을 예정
  - How it works
    메모리 안에서 로그 저장하는 자료구조
    같은 key값에 대해 여러 번 PUT 연산이 일어나면 그 중 마지막 것만 남김 (어차피 최종 결과가 중요하므로)
    MemTable이 가득 차면, SSTable로 옮김
    SSTable로 옮길 때는, **key가 증가하는 순으로** PUT(key101, abc)처럼 기록함
    이 한 뭉탱이를 Disk에 저장, SSTable끼리는 시간순으로
    Summary Table이라는, SSTable의 메타데이터를 담은 정보를 추가로 저장
    Read의 경우, MemTable에 있으면 그냥 가져오고, 없으면 SSTable에서 최신 것부터 찾음
    SSTable이 많아지면 계산이 비싸지므로 compaction을 해야 함 -> query, compaction 간의 tradeoff가 있음
  - SSTable Compaction
    같은 SST 내에서는 key 순이므로, SST 둘을 합칠 때에는 two pointer 알고리즘처럼 하면 됨
    언제 합침 그래서??
    - Leveled Compaction
      size가 다른 여러 level을 관리, level 0을 빼고는 key 범위가 겹치지 않도록 함
      한 level이 꽉 차면 compaction을 진행하고 결과를 다음 level에 추가
      level 1부터는 key 찾기가 편하므로 read-heavy workload에 유리
    - Universal Compaction
      level 나누지 않음, 겹치는 게 너무 많을 때 compaction 진행
      insert-heavy workload, time-oriented query에 유리
