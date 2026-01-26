---
title: 01 Relational Model & Algebra
author: haram
layout: post
---
## Database Systems Background
* Database / Database system
	* **Database**: 현실 세계의 일부를 모델링한, 서로 연관된 data의 organized collection.
	* **DBMS**: 그 database를 create / query / update / manage 해주는 software
	  > A database management system is software that allows applicaions to **store** **and analyze information in a database.**
	* 그냥 csv를 쓰면?
		* Data Integrity
		* Implementation
		* Duarability
	* 데이터베이스는 직접 어플에 작성되는 것이 아님. 데이터 관리 시스템을 사용해야 함. 
		* 앱은 **physical storage detail** 을 모르는 채로
		  DBMS가 상황에 맞게 query의 의미는 유지하면서 storage / indexing / execution plan을 바꿀 수 있도록 유지 가능.

## Schema
* 특정 데이터 집합의 기술. 데이터 모델을 위한 구조를 정의한다.

## Common Data Models
* **Relational** < 대체로 이걸 씀
* Key/Value < 간단한 앱/캐싱
* **Graph** < NoSQL
* **Document / JSON / XML / Object**  < NoSQL
* **Wide-Colum / Column-family**  < NoSQL
* Array (Vector, Matrix, Tensor) < ML / Science
* Hirarchical(Obsolete/Lagacy/Rare)
* Network(Obsolete/Lagacy/Rare)
* Multi-Value(Obsolete/Lagacy/Rare)

## Relational Model

* 관계형 모델은 사용자/애플리케이션을 저수준 데이터 표현으로부터 분리하는 데이터 독립성을 제공한다. 사용자는 고수준의 애플리케이션 로직만 관여하면 되는 형태로 서비스를 만들 수 있다. DBMS는 운영 환경, DB 내용, 워크로드에 따라 저장 레이아웃을 최적화하며, 이러한 요소가 변경되면 다시 최적화한다. (re-optimize) 따라서 **물리적 데이터의 변화가 애플리케이션을 변경시키지 않는다.**
### (1) Structure(구조)

- 물리적 표현과 독립적으로, 관계와 그 내용을 정의한다. 각 관계는 속성들의 집합을 가지며, 각 속성은 값들의 도메인을 가진다.
	- 데이터는 **Relation(table)** 로 표현
	- Relation은 **attribute(column)들의 집합 + tuple(row)들의 집합
	- 보통 relation은 **unordered set** 관점(순서 없음)
    
### (2) Integrity(무결성)

- 데이터베이스의 내용이 특정 제약을 만족하도록 보장한다. 예를 들어, "사람의 나이는 음수가 될 수 없다"같은 제약이 있다.
	- **Constraints**
	    - NOT NULL
	    - UNIQUE KEY
	    - FOREIGN KEY 
	    - CHECK
        
### (3) Manipulation(조작)

- 관계를 통해 데이터베이스의 내용을 조회하거나 수정하기 위한 선언적 API. 프로그래머는 원하는 결과만 지정하고, DB 시스템이 가장 효율적인 질의 실행 계획을 결정해 수행한다.
	- 집합 연산과 비슷하게 query, update를 할 수 있음
	- **Declarative** (> SQL)

## Relation / Tuple

* **Relation:** 엔티티를 나타내는 속성들 간의 관계를 담는 순서 없는 집합.(unordered set)
  관계는 순서가 없기 때문에 DBMS는 최적화를 위해 원하는 방식으로 저장할 수 있다. primary key가 다르기만 하다면 관계 안에 중복된 원소가 존재할 수 있다.
* **Tuple:** 관계 안에서의 속성 값들의 집합. (또는 그 도메인)
  과거에는 값이 원자적 또는 스칼라여야 했지만, 이제는 리스트나 중첩 데이터 구조도 값이 될 수 있다. 모든 속성은 특별한 값인 NULL을 가질 수 있는데, 이는 해당 튜플에서 그 속성이 정의되지 않았음을 의미한다.

## Primary Key / Foreign Key
* **Primary Key(PK):** 테이블에서 하나의 튜플을 유일하게 식별한다. 어떤 DBMS는 기본키를 정의하지 않으면 내부 기본키를 자동으로 만들기도 한다. 또한 identity column을 통해 유일한 기본키를 자동 생성할 수 있다.
* **Foreign Key(FK):** 한 관계의 어떤 속성이 다른 관계의 한 튜플에 대응되도록 지정한다. 일반적으로 외래키는 다른 테이블의 기본키를 가리키거나(=값이 같거나) 한다.

* ex)
	- **IDENTITY** (standard)
	- **SEQUENCE** (PostgreSQL/Oracle)
	- **AUTO_INCREMENT** (MySQL)


---
## DMLs: Data Manipulation Languages
* **DML**은 DBMS가 애플리케이션에 제공하는 API를 뜻하며, 데이터베이스로부터 정보를 **저장**하고 **조회**하는 데 사용된다. 데이터베이스를 조작하는 언어는 크게 두 부류가 있다.
	* **절차적(Procedural)**: 원하는 결과를 얻기 위해 DBMS가 따라야 할 (고수준의) **실행 전략**을 질의가 직접 지정한다.
	* **비절차적(Non-Procedural, 선언적/Declarative):** 무슨 데이터가 필요한지(what)만 지정하고, 어떻게 찾는지(how)는 지정하지 않는다.

### Relational Algebra

* SQL(선언적 언어)에서는 무엇을 계산할지만 표현하고 어떻게 계산할지는 지정하지 않는다. DBMS가 질의 최적화(Query Optimization)를 통해 최선의 실행 전략을 찾는다. 이런 강력한 추상화 덕분에 SQL은 관계형 DBMS에서 질의를 작성하는 사실상 표준(de facto standard)이 되었고, 사용자는 DBMS 내부를 몰라도 가장 효율적인 방식으로 질의할 수 있다.

* NULL: 아무것도 추론할 수 없음(정의되지 않음)

* **Select (σ)**: 한 관계를 입력으로 받아, 선택 조건(predicate)을 만족하는 튜플의 부분집합을 출력한다. 이 predicate는 필터처럼 동작하며, 여러 predicate는 AND/OR 같은 논리 결합으로 조합할 수 있다.
	* $\sigma_{p}(R)$
	* e.g. $\sigma_{a\_id='a2'}(R)$
	* SQL
	* ```sql
	  SELECT *
	  FROM R
	  WHERE a_id = 'a2';
	  ```
- **Projection (π)**: 한 관계를 입력으로 받아, 지정한 속성(attribute)만 포함하는 튜플들로 이루어진 관계를 출력한다. 입력 관계에서 속성의 순서를 바꾸거나, 값을 조작하는 것도 가능하다.
	- $\pi_{A_1, A_2, \dots, A_n}(R)$
	- $\pi_{b_id-100,\ a_id}\big(\sigma_{a_id='a2'}(R)\big)$
	- ``` sql
	  SELECT b_id - 100, a_id
	  FROM R
	  WHERE a_id = 'a2';
	  ```
- **Union (∪)**: 두 관계를 입력으로 받아, 둘 중 하나 이상에 등장하는 모든 튜플을 포함하는 관계를 출력한다.
	- `UNION`은 중복 제거, `UNION ALL`은 중복 유지
	- $(R \cup S)$
	- ```sql
	  (SELECT * FROM R)
	  UNION ALL
	  (SELECT * FROM S);
	  ```
- **Intersection (∩)**: 두 관계를 입력으로 받아, 두 관계 모두에 등장하는 튜플을 포함하는 관계를 출력한다. 두 입력 관계는 완전히 같은 속성을 가져야 한다.
	- $(R \cap S)$
	- ```sql
	  (SELECT * FROM R)
	  INTERSECT
	  (SELECT * FROM S);
	  ```
- **Difference (−)**: 두 관계를 입력으로 받아, 첫 번째에는 있지만 두 번째에는 없는 모든 튜플을 포함하는 관계를 출력한다.
	- $(R - S)$
	- ```sql
	  (SELECT * FROM R)
	  EXCEPT
	  (SELECT * FROM S);
	  ```
- **Product (×)**: 두 관계를 입력으로 받아, 입력 관계의 튜플들로 만들 수 있는 모든 조합을 포함하는 관계를 출력한다.
	- $(R \times S)$
	- ```sql
	  (SELECT * FROM R) CROSS JOIN (SELECT * FROM S);
	  -- 또는
	  SELECT * FROM R, S;
	  ```
- **Join (⋈)**: 두 관계를 입력으로 받아, 두 관계의 튜플을 결합한 결과 중에서 두 관계가 공유하는 각 속성 값이 동일한 조합들로 이루어진 관계를 출력한다.
	- $(R \bowtie S)$
	- ```sql
	  SELECT *
	  FROM R JOIN S USING (ATTRIBUTE1, ATTRIBUTE2, ...);
	  ```
