---
title: Nested loop Join
date: 2024-05-04 17:56:00 +09:00
categories: [database, DBMS]
tags:
  [
    DBMS,
    NL join,
    Oracle
  ]
  
---

# Nested Loop Join

## 목차

1. [메커니즘](#1-메커니즘)
2. [특징](#2-특징)
3. [실행 예시](#2-실행-예시)
4. 튜닝
  1. prefetch
  2. batch IO
  3. buffer pinning


## 1. 메커니즘

조인하는 두 테이블에 대해서, outer table, inner table을 정한 후

outer table에서 scan 해온 블럭 내에 row를 순회하면서 row by row로 조인키 조건에 만족하는 inner table 내에 row들과 결합하게 됩니다.
- 프로그래밍에서 **nested loop 문**을 생각하면 됩니다.

## 2. 특징

1. **random access 방식 위주**의 조인 방식
  - 대량의 데이터를 가진 테이블로 조인할 시, 비효율적이다.
2. 부분범위처리 가능한 조인 방식
  - 빠른 응답 속도를 낼 수 있음
3. 인덱스 구성 전략이 중요
  - 인덱스 존재 유무, 컬럼 구성에 따라 성능이 좌우된다.
    - inner table에 인덱스가 존재하고 index key를 join key로 사용할 때 outer table의 하나의 row가 inner table 전체와 조인을 시도하는 짓을 방지할 수 있다.
    - 또한, sql의 where절 predicate들의 column들이 index column들로 적절히 구성되어 있을 때 index [range, unique, ...] scan들로 table access량을 줄일 수 있다.
    

>>  OLTP 환경에서 적합


## 3. 실행 예시


index을 이용한 nl join을 시작으로 튜닝을 시작하고자 한다.
- Oracle 19c
근데,,, 

```sql
------------------------------------------------
| Id  | Operation			                         |
------------------------------------------------
|   0 | SELECT STATEMENT		                   |	 
|   1 |  SORT ORDER BY			                   |	 
|   3 |   NESTED LOOPS 		                     |	 
|*  4 |    TABLE ACCESS BY INDEX ROWID         |
|*  5 |     INDEX RANGE SCAN		               |
|*  2 |    TABLE ACCESS BY INDEX ROWID         |
|*  6 |     INDEX RANGE SCAN		               |
------------------------------------------------
```
와 같은 플랜을 재현하고 싶었는데, Oracle 11c부터 시작된 batch IO와 9i에서 시작된 prefetch 기능으로 인해, 원하는 대로 재현이 잘 안됐다.


```sql
alter system set "_table_lookup_prefetch_size"=0  scope=spfile;

drop table outer purge;
drop table inner purge;
create table outer (c1 number, c2 number, c3 number);

create index i_outer on outer(c2);

create table inner (c1 number, c2 number, c3 number);

create index i_inner on inner(c1);

insert into outer select object_id, namespace, data_object_id from dba_objects;
insert into inner select object_id, namespace, data_object_id from dba_objects;
commit;

select /*+ n ordered use_nl(i) index_asc(o) index_asc(i) opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0')*/ o.*, i.* from outer o, inner i where i.c1 = o.c1 and o.c2 < 1 and i.c3 < 1000;

```




출처

https://positivemh.tistory.com/1026