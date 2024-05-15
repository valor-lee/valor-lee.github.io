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
3. [실행 예시](#3-실행-예시)
4. [튜닝](#4-튜닝)
    1. prefetch
    2. batch IO
    3. buffer pinning


## 1. 메커니즘

조인하는 두 테이블에 대해서, outer table, inner table을 정한 후

outer table에서 scan 해온 블럭 내에 row를 순회하면서 

row by row로 조인키 조건에 만족하는 inner table 내에 row들과 결합하게 됩니다.
- 프로그래밍에서 **nested loop 문**을 생각하면 됩니다.

## 2. 특징

1. **random access 방식 위주**의 조인 방식
  - 대량의 데이터를 가진 테이블로 조인할 시, 비효율적이다.
2. **부분범위처리 가능**한 조인 방식
  - 빠른 응답 속도를 낼 수 있음
3. 인덱스 구성 전략이 중요
  - 인덱스 존재 유무, 컬럼 구성에 따라 성능이 좌우된다.
    - inner table에 인덱스가 존재하고 index key를 join key로 사용할 때 outer table의 하나의 row가 inner table 전체와 조인을 시도하는 짓을 방지할 수 있다.
    - 또한, sql의 where절 predicate들의 column들이 index column들로 적절히 구성되어 있을 때 index [range, unique, ...] scan들로 table access량을 줄일 수 있다.
    

>>  OLTP 환경에서 소량의 테이블 간 join 시 적합한 방식


## 3. 실행 예시

index을 이용한 nl join을 시작으로 튜닝을 시작하고자 한다.
- Oracle 19c

근데,,, 

```sql
------------------------------------------------
| Id  | Operation
------------------------------------------------
|   0 | SELECT STATEMENT	 
|   1 |  SORT ORDER BY	 
|   3 |   NESTED LOOPS 
|*  4 |    TABLE ACCESS BY INDEX ROWID
|*  5 |     INDEX RANGE SCAN
|*  2 |    TABLE ACCESS BY INDEX ROWID
|*  6 |     INDEX RANGE SCAN
------------------------------------------------
```
와 같은 플랜을 재현하고 싶었는데, Oracle 11c부터 시작된 batch IO와 9i에서 시작된 prefetch 기능으로 인해, 원하는 대로 재현이 잘 안됐다.

아래와 같은 조치를 통해 플랜을 재현할 수 있었다.

- _table_lookup_prefetch_size=0(default 40)으로 설정
  - oracle에서 테이블 prefetch 갯수를 제어하는 init param으로, session scope으로 변경이 되지 않아서 scope=spfile 구문을 추가하여 변경했다.
  - 9.2.0 버전부터 생긴 hidden parameter
- opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0') 힌트
  - 옵티마이저 기능을 특정 버전으로 제한함.
  - table access by rowid batched 기능이 힌트로도 꺼지지 않아서 opt version을 낮췄음
- _db_cache_pre_warm
  - buffer cache prewarm 기능으로 physical read prefetch가 발생했음
  - 10.1.0 버전부터 생긴 hidden parameter

```sql
alter system set "_table_lookup_prefetch_size"=0  scope=spfile;
alter system set "_db_cache_pre_warm"=false scope=spfile;

-- reboot

drop table outer purge;
drop table inner purge;
create table outer (c1 number, c2 number, c3 number);

create index i_outer on outer(c2);

create table inner (c1 number, c2 number, c3 number);

create index i_inner on inner(c1);

insert into outer select object_id, namespace, data_object_id from dba_objects;
insert into inner select object_id, namespace, data_object_id from dba_objects;
commit;

set linesize 130
-- 통계수집 수준 제어 (basic, typical, all)
alter session set statistics_level=all;

select /*+ ordered use_nl(i) index(o) index(i) opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0')*/ count(*) from outer o, inner i where i.c1 = o.c1 and o.c2 = 4 and o.c3 < 100 and i.c3 < 50;

select * from dbms_xplan.display_cursor(null, null, 'advanced allstats last -alias -projection');
-- 직전 쿼리에 대한 플랜, 자세한 execution stat, 별칭과 projection 정보 생략하여 출력
------------------------------------------------------------------------------
| Id  | Operation		                    | Name    | A-Rows |  Buffers |
------------------------------------------------------------------------------
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT	              | 	      |	     1 |      68 |
|   1 |  SORT AGGREGATE 	              | 	      |	     1 |      68 |
|   2 |   NESTED LOOPS		              | 	      |	    25 |      68 |
|*  3 |    TABLE ACCESS BY INDEX ROWID    |   OUTER	  |	    54 |      59 |
|*  4 |     INDEX RANGE SCAN	          |   I_OUTER |   3341 |      12 |
|*  5 |    TABLE ACCESS BY INDEX ROWID    |     INNER |	    25 |       9 |
|*  6 |     INDEX RANGE SCAN	          |   I_INNER |	    54 |	   8 |
------------------------------------------------------------------------------



PLAN_TABLE_OUTPUT
---------------------------------------------------------------

  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('9.2.0')
      DB_VERSION('19.1.0')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      INDEX_RS_ASC(@"SEL$1" "O"@"SEL$1" ("OUTER"."C2"))
      INDEX_RS_ASC(@"SEL$1" "I"@"SEL$1" ("INNER"."C1"))
      LEADING(@"SEL$1" "O"@"SEL$1" "I"@"SEL$1")
      USE_NL(@"SEL$1" "I"@"SEL$1")
      END_OUTLINE_DATA
  */

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - filter("O"."C3"<100)
   4 - access("O"."C2"=4)
   5 - filter("I"."C3"<1000)
   6 - access("I"."C1"="O"."C1")


select name, value from v$statname vs, v$mystat vm where vs.statistic# = vm.statistic# and vs.name like 'physical reads%';
--  physical read 관련 stat 출력

NAME								                                  VALUE
---------------------------------------------------------------- ----------
physical reads								                            117
physical reads cache							                        117
physical reads direct							  			              0
physical reads direct temporary tablespace				                  0
physical reads cache prefetch						                      0
physical reads prefetch warmup						                      0
physical reads retry corrupt						                      0
physical reads direct (lob)						  			              0
physical reads for flashback new					  			          0
physical reads cache for securefile flashback block new 		          0
physical reads direct for securefile flashback block new		          0
physical reads for data transfer					                      0
```

PLAN_TABLE_OUTPUT
- 원하는 형태의 plan이 재현되었다.
  - BEGIN_OUTLINE_DATA : END_OUTLINE_DATA 사이에 사용되는 힌트들이 sql문이 실행될 때마다 동일한 실행계획을 유지하도록 함
  - IGNORE_OPTIM_EMBEDDED_HINTS : 이 SQL 문 내에 포함된 다른 모든 힌트를 무시하도록 지시한다. sql profile 기능을 통해 특정 sql을 의도한 플랜으로 풀리도록 고정하는 데 사용할 수 있다.
    - [참고](https://hrjeong.tistory.com/288)

Predicate Information
- access 순서는 아래와 같다.
  1. outer table index scan (o.c2 = 4)
  2. outer table random access (o.c3 < 100)
  3. inner table index scan (i.c1 = o.c1)
  4. inner table random access (i.c3 < 1000)
- 이처럼, nl join의 경우 outer table에서 선택되는 row 수가 많을 수록 join의 부하가 커진다.

physical reads
- 의도한 대로 prefetch count가 0임을 볼 수 있음

## 4. 튜닝

### 튜닝 순서

1. OLTP 환경에서 join 튜닝은 NL join 튜닝부터 시작한다.
2. NL join의 매커니즘을 바탕으로 각 단계에서 과도한 비용이 발생하는 부분을 파악한다.
3. 튜닝 방법을 고려한다.
    1. 조인 순서 변경
    2. [index column 추가](#outer-table-random-access-줄이기)
    3. [다른 인덱스 사용(복합 인덱스 컬럼 순서 조정)](#index-range-scan-선두컬럼이-범위조건으로-사용될-경우)
    4. 다른 join 방식 고려


### outer table random access 줄이기

위 플랜 4번 노드에서 row 수가 3341개 이기 때문에 table로 3341번 single block IO가 발생했음을 알 수 있다.

하지만, outer table에서 필터링 이후 54개가 남았고, **불필요한 table access가 많이 발생했다는 것을 알 수 있다.**

>> 여기서 가능하다면 i_outer 인덱스에 outer table 필터링 컬럼을 추가하는 것을 고려할 수 있음

```sql

drop index i_outer;
-- add the outer table filter predicate column 
create index i_outer on outer (c2, c3);

select /*+ ordered use_nl(i) index(o) index(i) opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0')*/ count(*) from outer o, inner i where i.c1 = o.c1 and o.c2 = 4 and o.c3 < 100 and i.c3 < 50;

select * from dbms_xplan.display_cursor(null, null, 'advanced allstats last -alias -projection');

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------
| Id  | Operation		                    | Name    | A-Rows |  Buffers |
------------------------------------------------------------------------------
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------
...
|   2 |   NESTED LOOPS		              | 	      |	    25 |       9 |
|*  3 |    TABLE ACCESS BY INDEX ROWID    |   OUTER	  |	    54 |       3 |
|*  4 |     INDEX RANGE SCAN	          |   I_OUTER |     54 |       2 |
|*  5 |    TABLE ACCESS BY INDEX ROWID    |     INNER |	    25 |       9 |
|*  6 |     INDEX RANGE SCAN	          |   I_INNER |	    54 |	   8 |
------------------------------------------------------------------------------


Predicate Information (identified by operation id):
---------------------------------------------------
   4 - access("O"."C2"=4 AND "O"."C3"<100)
   5 - filter("I"."C3"<50)
   6 - access("I"."C1"="O"."C1")


select name, value from v$statname vs, v$mystat vm where vs.statistic# = vm.statistic# and vs.name like 'physical reads%' or vs.name like '%read%parallel%';

NAME								                                  VALUE
---------------------------------------------------------------- ----------
physical reads								                             59
physical reads cache							                         59
physical reads direct							  			              0
physical reads direct temporary tablespace				                  0
physical reads cache prefetch						                      0
physical reads prefetch warmup						                      0
physical reads retry corrupt						                      0
physical reads direct (lob)						  			              0
physical reads for flashback new					  			          0
physical reads cache for securefile flashback block new 		          0
physical reads direct for securefile flashback block new		          0
physical reads for data transfer					                      0
```

- 여기서 **i_outer의 컬럼 조합의 순서**가 중요하다.
  - outer table predicate를 살펴보면 o.c2의 경우 등호조건, o.c3는 부등호 조건이다.
  - c3를 선두 컬럼으로 순서를 구성할 경우 부등호 조건으로 인해 두 개의 컬럼을 predicate로 사용하더라도, 인덱스 구조 및 정렬에 의해서 index leaf block의 scan량을 줄일 수 없다. [예시](#index-range-scan-선두컬럼이-범위조건으로-사용될-경우)
  - 그래서 등호 조건이 사용된 c2 컬럼이 선두가 되도록 컬럼 순서를 정했다.
- 4번 노드에서 index range scan의 일량 자체를 줄임으로써 random access 횟수를 줄일 수 있었다.
- table access by index rowid 의 경우 single block IO 방식으로 진행되기 떄문에 3번 노드의 실행 횟수를 줄인다는 것은 비효율을 크게 줄였음을 의미한다.




### index range scan 선두컬럼이 범위조건으로 사용될 경우

```sql
drop index i_outer;
-- index column 순서 변경
create index i_outer on outer (c3, c2);

select /*+ ordered use_nl(i) index(o) index(i) opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0')*/ count(*) from outer o, inner i where i.c1 = o.c1 and o.c2 = 4 and o.c3 < 100 and i.c3 < 50;

select * from dbms_xplan.display_cursor(null, null, 'advanced allstats last -alias -projection');
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------
| Id  | Operation		                    | Name    | A-Rows |  Buffers |
------------------------------------------------------------------------------
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------
...
|   2 |   NESTED LOOPS		              | 	      |	    25 |       9 |
|*  3 |    TABLE ACCESS BY INDEX ROWID    |   OUTER	  |	    54 |       3 |
|*  4 |     INDEX RANGE SCAN	          |   I_OUTER |     54 |       2 |
|*  5 |    TABLE ACCESS BY INDEX ROWID    |     INNER |	    25 |       9 |
|*  6 |     INDEX RANGE SCAN	          |   I_INNER |	    54 |	   8 |
------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   4 - access("O"."C2"=4 AND "O"."C3"<100)
       filter("O"."C2"=4)
   5 - filter("I"."C3"<50)
   6 - access("I"."C1"="O"."C1")

```

- 인덱스 컬럼을 c3, c2 순서(sql에서 범위 조건절에 사용된 컬럼이 선두가 되도록)로 변경했다.
  - 하지만, 스캔한 buffer가 2로 동일했는데, 이는 c2 = 4 and c3 < 100 을 만족하는 row가 54개 밖에 되지 않아서 그런 것이다.
- predicate information을 보면 c2가 index range scan에서 인덱스 수평 탐색 filter 조건으로 사용되었다.
  - 이는 c3 < 100 이라는 조건에서 c2의 distinct value가 많다면 불필요한 리프블럭을 읽어야 하는 비효율이 발생할 수 있다.
  - 다행히? 지금의 데이터 구성으로는 row 수나 buffer 수를 봤을 때 비효율이 보이지 않는다.


### 조인 순서 조정

- 조인 순서를 조정해서 효과를 볼 수 있는 케이스는 대표적으로 각 테이블의 predicate로 선택되는 row수가 차이가 날 때이다.
  - 아래 케이스처럼 데이터의 cardinality가 차이 날 때(inner table의 selectivity가 낮을 때) 조인 순서를 변경하면 된다.
  - block IO 개수가 2배 차이나는 것을 확인할 수 있다.
  - 단순히 조인 순서만 변경한 것이기 때문에 index column 조합 등의 튜닝이 추가로 요구되어 보기이긴 하다.
- 조인 순서를 변경하여도 소득이 없거나 성능이 만족스럽지 않은 경우, 다른 조인 방식을 적용할 고려를 해봐야 한다.

```sql
drop table outer purge;
drop table inner purge;
create table outer (c1 number, c2 number, c3 number);

create index i_outer on outer (c2, c3);

create table inner (c1 number, c2 number, c3 number);

create index i_inner on inner(c1);

insert into outer select ceil(dbms_random.value(0, 100)), ceil(dbms_random.value(0, 4)), ceil(dbms_random.value(0, 100)) from dual connect by level < 4000;
commit;


-- inner table의 selectivity를 낮추기 위해 distinct value range를 늘리고 중복도를 낮춤
insert into inner select ceil(dbms_random.value(0, 100)), ceil(dbms_random.value(0, 4)), ceil(dbms_random.value(0, 10000)) from dual connect by level < 4000;



select /*+ ordered use_nl(i) index(o) index(i) opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0')*/ count(*) from outer o, inner i where i.c1 = o.c1 and o.c2 = 4 and o.c3 < 100 and i.c2 = 4 and i.c3 < 100;

PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------
| Id  | Operation		                 |    Name  |    A-Rows |  Buffers|
---------------------------------------------------------------------------
...
|   2 |   NESTED LOOPS		            | 	        |		 48 |    9572 |
|   3 |    TABLE ACCESS BY INDEX ROWID  |    OUTER	|	    923 |     559 |
|*  4 |     INDEX RANGE SCAN	        |   I_OUTER |       923 |	    9 |
|*  5 |    TABLE ACCESS BY INDEX ROWID  |     INNER	|	     48 |    9013 |
|*  6 |     INDEX RANGE SCAN	        |   I_INNER |	  36945 |     994 |
---------------------------------------------------------------------------

select /*+ leading(i o) use_nl(o) index(i) opt_param('OPTIMIZER_FEATURES_ENABLE', '9.2.0')*/ count(*) from outer o, inner i where i.c1 = o.c1 and o.c2 = 4 and o.c3 < 100 and i.c2 = 4 and i.c3 < 100;

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------
| Id  | Operation		                |    Name   |   A-Rows | Buffers |
--------------------------------------------------------------------------
...
|   2 |   NESTED LOOPS		            |       	|     	48 |    4202 |
|*  3 |    TABLE ACCESS BY INDEX ROWID  |   INNER	|     	 6 |     876 |
|   4 |     INDEX FULL SCAN	            | I_INNER   |     3999 |       9 |
|*  5 |    TABLE ACCESS BY INDEX ROWID  |   OUTER	|     	48 |    3326 |
|*  6 |     INDEX RANGE SCAN	        | I_OUTER   |     5538 |      26 |
--------------------------------------------------------------------------

```




출처

오라클 성능 고도화 원리와 해법 I

https://positivemh.tistory.com/1026