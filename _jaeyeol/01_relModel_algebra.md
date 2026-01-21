---
title: 01 Relational Model & Algebra
author: jaeyeol
layout: post
---

# Definition
**Database**: real-world 데이터를 모델링하는 데이터 뭉탱이  
-> 형태가 정해져 있지 않으며 사실 우리가 이용하는 것들의 대부분이 DB라고 볼 수 있음  
-> 현재 컴퓨터 애플리케이션의 핵심 구성요소  

CSV File에 데이터를 저장한다고 해보자. 어떤 문제가 생길까?  
데이터 하나 찾는 것도 오래 걸림, type 없음, 하드코딩, 일관성 없음, ...  

**DBMS** (DataBase Management System): DB에 정보를 저장, 분석할 수 있게 해주는 시스템  
-> `Data Model`을 주게 됨 (`DB에 데이터를 어떻게 나타낼 것인가` 를 추상화한 것)  
Data Model의 예시  
- Relational Model - 가장 자주 사용되는 모델
- Key/Value - Caching 등에 사용
- Graph, Document/JSON/XML/Object, Wide-column/Column-family - NoSQL
- Array (Vector, Matrix, Tensor) - ML/DS
- Hierarchical/Network/Semantic/E-R - Legacy

**Schema**: data model이 주어졌을 때, 특정 DB에 대한 설명 (blueprint와도 같음)  

# Early DBMS
1960년대: 개발자들이 procedural code 작성 (파일 경로도 골라야 되고 겁나복잡함)  
1970년대 Relational Model 등장  
Structure: `논리적인` 레벨에서 데이터 간의 connection 표현  
Integrity: DB의 내용물이 특정 제한 (constraint)를 만족하는 것을 보장  
Manipulation: DB 내용에 접근, 수정할 수 있는 API  
-> 논리적인 레벨과 물리적인 레벨은 독립적임 (SQL query가 주어졌을 때 해당 쿼리를 어떻게 수행할지는 DBMS가 설계)  

# Concepts related to RDB
Relation: 테이블 개념 생각하면 됨  
Tuple: Relation의 원소 하나  
Primary Key: 특정 relation 안에서 모든 tuple을 고유하게 식별할 수 있는 attribute (unique id, ...)  
Forgien Key: relation 간의 many-to-many correspondence를 처리하기위해 특정 relation의 특정 attribute를 다른 relation의 것에 매핑  
Constraint: 허용되는 값의 종류를 지정  
DML: 정보 저장, 수정을 위한 API  
- Procedural DML: 쿼리 수행 방법을 정확히 정의 (relational algebra)
- Declarative DML: 쿼리가 무슨 데이터를 가져올지만 정하고 어떻게는 정하지 않음 (relational calculus)

# Relational Algebra
여러 가지 연산이 정의됨  
select: 특정 filter와 함께 정의, filter에 명시된 조건을 만족시키는 부분집합을 반환  
project: 출력할 column의 순서를 바꾸거나 일부만 표시하거나 그런 걸 할 수 있음  
union, intersection, difference: 합집합, 교집합, 차집합에 대응, attribute name이 완전히 같은 relation 사이에서만 정의됨  
cartesian product: A에 속한 모든 a, B에 속한 모든 b에 대해 가능한 모든 (a, b) 조합을 반환  
join: cartesian product + filter  
이외에도 rename, assignment, duplicate elimination, aggregation, sorting, division 등의 연산자가 있음
이때, 서로 다른 연산이 논리적으로 같은 출력 결과를 뱉을 수 있음 -> procedural
