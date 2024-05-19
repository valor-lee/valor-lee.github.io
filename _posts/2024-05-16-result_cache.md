---
title: result cache
date: 2024-05-16 22:10:00 +09:00
categories: [database, DBMS]
tags:
  [
    DBMS,
    result_cache,
    Oracle
  ]
---

# result cache

## 목차
  
- [result cache](#result-cache)
  - [목차](#목차)
  - [1. 정의](#1-정의)
  - [2. 특징](#2-특징)
  - [3. 메커니즘](#3-메커니즘)
  - [4. 실행 예시](#4-실행-예시)
  - [5. 튜닝](#5-튜닝)
    - [인라인 뷰 내에서 result cache](#인라인-뷰-내에서-result-cache)
    - [with구문에서 result cache](#with구문에서-result-cache)
    - [union all로 부분 result cache](#union-all로-부분-result-cache)


## 1. 정의

쿼리나 PL/SQL 수행 결과를 shared memory에 캐싱해두고, 동일한 쿼리가 수행될 때 block IO나 연산 없이 결과값을 반환해주는 기능이다.

## 2. 특징

수행한 결과를 **shared memory 영역 내에 저장**한다.
- SQL 결과와 PL/SQL 결과를 저장하는 영역을 나누어 관리한다.
- shared memory 영역이기 때문에 result cache 기능에서 latch mechanism이 필요하다.

dml이 자주 발생하지 않는 테이블을 대상으로 반복 수행 요청이 많은 쿼리에 적합한 기능
- 데이터 정합성을 맞추기 위해 dml이 발생한 object와 관련된 result cache는 invalid state로 바뀌고 재갱신하는 비효율이 발생함
- **bind parameter**를 사용할 경우 전달되는 **값이 바뀌면** 새로운 result set을 저장하게 된다. 즉, **caching 효과가 없음**

몇몇 object를 포함한 sql은 caching이 불가능하다.
- data dictionary object, temporary table, date 또는 time 관련 object, non-constant variable

`v$result_cache_objects`

- 캐싱되어 있는 object의 목록을 보여준다.


**관련 파라미터**
- `result_cache_mode`(manual(default) / force)
  - manual : result_cache 힌트를 포함한 sql의 result만 캐싱한다.
  - force : no_result_cache 힌트를 포함한 sql 외에 sql의 result를 캐싱한다.  
  - alter [session/system] set result_cache_mode='[manual/force]';
- `result_cache_max_size`
  - SGA에서 result cache가 사용할 영역을 byte로 지정한다.
  - 0 으로 지정할 시 result cache 기능을 사용하지 않는다.
  - Oracle 버전에 따라 memory 영역 관리 방식이 다양한데, 결국 shared pool의 75%를 넘지 못하도록 관리한다고 한다.
    - ~~75%는 너무 크지 않은가..?~~
- `result_cache_max_result`
  - 하나의 result set이 result cache memory를 얼마나 차지할 수 있는 최대 크기 지정(%)
- `result_cache_remote_expiration`
  - remote object 관련 result set을 몇 분동안 캐시 유지할 건지 설정하는 파라미터
  - **remote object** : db link를 통해 연결된 다른 db의 테이블이나 뷰를 의미한다.


## 3. 메커니즘

1. result_cache_mode를 켜주고, 결과를 캐싱하고 싶은 sql에 `RESULT_CACHE` 힌트를 작성한다.
2. 서버에서 요청에 대해서 SGA의 shared pool의 result cache 메모리에서 찾아본다.
   1. hit : cache 되어 있는 결과 집합을 리턴
   2. miss : 쿼리를 수행하고 결과 리턴, result cache 메모리에 캐싱한다.


## 4. 실행 예시

```sql
SQL> show parameter result_cache_mode

NAME				                TYPE	 VALUE
----------------------------------------------------
result_cache_mode		            string	 MANUAL

SQL> show parameter result_cache_max_size

NAME				                TYPE	 VALUE
----------------------------------------------------
result_cache_max_size           big integer 12128K

SQL> show parameter result_cache_max_result

NAME				                   TYPE	 VALUE
----------------------------------------------------
result_cache_max_result 	          integer	 5


SQL> show parameter result_cache_remote_expiration

NAME				                TYPE	 VALUE
----------------------------------------------------
result_cache_remote_expiration	     integer	 0

```
- result_cache 관련 파라미터들은 위와 같은 상황에서 테스트를 진행했다.

```sql
SQL> select /*+result_cache*/ * from all_tables;

...
2218 rows selected.


SQL> select name, scan_count from v$result_cache_objects where name like '%all_tables%';

no rows selected

```

- result_cache 특징 중 하나로, dd object들을 참조하는 쿼리는 캐싱되지 않는 것을 확인할 수 있다.

```sql
drop table result_cache;

create table result_cache (c1 number, c2 number);

insert into result_cache select level, level from dual connect by level < 100;

commit;

select /*+ result_cache */ count(*) from result_cache;

  COUNT(*)
----------
	99

Execution Plan
------------------------------------------------------------------
| Id  |     Operation	    |               Name	     | Rows  |
------------------------------------------------------------------
|   0 | SELECT STATEMENT    |				             |     1 |
|   1 |  RESULT CACHE	    | 4dkykmc8sj7076b9pv1fa3grr7 |     1 |
|   2 |   SORT AGGREGATE    |				             |     1 |
|   3 |    TABLE ACCESS FULL|       RESULT_CACHE		 |    99 |
------------------------------------------------------------------

select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACHE%';

ID  NAME													          TYPE       STATUS
--------------------------------------------------------------- -------------------------
22  .RESULT_CACHE										    Dependency    Published
23  select /*+ result_cache */ count(*) from result_cache		    Result    Published

```
- type이 result인 row는 published 상태 즉, 캐시 결과가 유효하다는 것을 뜻한다.
- result cache될 때 쿼리에 포함된 object를 dependency 타입으로 관리한다.
- 

```sql

insert into result_cache values (1, 1);

-- dml 직 후 cache status 변화
select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACHE%';

ID  NAME													          TYPE       STATUS
--------------------------------------------------------------- -------------------------
22  .RESULT_CACHE										    Dependency    Published
23  select /*+ result_cache */ count(*) from result_cache		    Result    Published


commit;

-- commit 이후 cache status 변화
select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACHE%';

ID  NAME													          TYPE       STATUS
--------------------------------------------------------------- -------------------------
22  .RESULT_CACHE										    Dependency    Published
23  select /*+ result_cache */ count(*) from result_cache		    Result      Invalid


select /*+ result_cache */ count(*) from result_cache;

  COUNT(*)
----------
	99

-- result cache invalid 이후 다시 cache 했을 때 status 변화
select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACH%';

ID  NAME													          TYPE       STATUS
--------------------------------------------------------------- -------------------------
22  .RESULT_CACHE										    Dependency    Published
23  select /*+ result_cache */ count(*) from result_cache		Result      Invalid
24  select /*+ result_cache */ count(*) from result_cache		Result    Published

```

- result cache에서 참조하는 object에 dml이 발생했을 때 result는 invalid 되는 것으로 확인된다.
- 또한, commit 시점에서 result cache가 invalid 상태가 되는 것을 확인할 수 있다.
- 동일한 쿼리를 재수행할 시, 23번 id의 result cache를 published 상태로 변경하는 것이 아닌, 새로운 cache를 생성한다.
  - 그래서, dml이 자주 발생하는 object를 참조하는 쿼리를 result cache에 사용한다면 상당한 비효율이 발생할 수 있음을 예상할 수 있었다.

```sql
drop table result_cache2;

create table result_cache2 as select level c1 from dual connect by level < 100;

select /*+ result_cache */ count(*) from result_cache o, result_cache2 i where o.c1 = i.c1;

  COUNT(*)
----------
       101

select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACH%';

	ID NAME 																	                        TYPE	  STATUS
------------------------------------------------------------------------------------------------------------------------------
	27 LWI.RESULT_CACHE2																                Dependency Published
	22 LWI.RESULT_CACHE																                    Dependency Published
	28 select /*+ result_cache */ count(*) from result_cache o, result_cache2 i where o.c1 = i.c1		Result	   Published

```
- 복수의 object를 참조하는 쿼리를 result cache할 경우, 당연히 모든 object에 대해서 dependency cache 한다.

```sql

drop table result_cache;

drop table result_cache2;


select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACH%';

	ID NAME 																	                        TYPE	  STATUS
----------------------------------------------------------------------------------------------------------------------------
	27 LWI.RESULT_CACHE2																                Dependency Invalid
	22 LWI.RESULT_CACHE																                    Dependency Invalid
	28 select /*+ result_cache */ count(*) from result_cache o, result_cache2 i where o.c1 = i.c1		Result	   Invalid

```

- 관련 object에 ddl이 발생할 때 dependency와 result가 invalid 상태가 된다.

## 5. 튜닝

### 인라인 뷰 내에서 result cache

```sql
drop table result_cache;
drop table result_cache2;

create table result_cache as select level c1 from dual connect by level < 100;

create table result_cache2 as select level c1 from dual connect by level < 100;

select * 
from 
    result_cache o,
    (select /*+ result_cache */ c1 + 1 as c2 from result_cache2 where c1 < 10) i 
where o.c1 = i.c2;

select id, name, type, status from v$result_cache_objects where name like '%result_cache2%' or name like '%RESULT_CACH%';

	ID NAME 																	       TYPE	      STATUS
------------------------------------------------------------------------------------------------------------
	29 LWI.RESULT_CACHE2															   Dependency Published
	30 select /*+ result_cache */ c1 + 1 as c2 from result_cache2 where c1 < 10		   Result	  Published

```

- inline view result만 cache만 되는 것을 확인할 수 있다.


### with구문에서 result cache

```sql
drop table result_cache;
drop table result_cache2;

create table result_cache as select mod(level, 10) c1, level c2 from dual connect by level < 100;

create table result_cache2 as select level c1 from dual connect by level < 100;

with with_view
as (
    select /*+ result_cache */ c1, sum(c2) as sub_sum from result_cache group by c1
)
select i.c1, o.sub_sum from with_view o, result_cache i where o.c1 = i.c1;

select id, name, type, status from v$result_cache_objects where name like '%result_cache%' or name like '%RESULT_CACH%';

	ID NAME 																	               TYPE	  STATUS
----------------------------------------------------------------------------------------------------------------
	31 LWI.RESULT_CACHE																     Dependency   Published
	1  select /*+ result_cache */ c1, sum(c2) as sub_sum from result_cache group by c1	     Result	  Published

```

### union all로 부분 result cache

```sql
drop table result_cache;

create table result_cache as select mod(level, 10) c1, level c2 from dual connect by level < 100;
commit;

select * from (
    select /*+ result_cache */ c1, sum(c2) as c2 from result_cache group by c1
    union all
    select * from result_cache
)
order by c1, c2;


Execution Plan
---------------------------------------------------------------------
| Id  | Operation	       | Name			                | Rows  |
---------------------------------------------------------------------
|   0 | SELECT STATEMENT       |			                |	109 |
|   1 |  SORT ORDER BY	       |			                |	109 |
|   2 |   VIEW		           |			                |	109 |
|   3 |    UNION-ALL	       |			                |	    |
|   4 |     RESULT CACHE       | 7atut8xus5yjxb97y9nvkg0art |	 10 |
|   5 |      HASH GROUP BY     |			                |	 10 |
|   6 |       TABLE ACCESS FULL| RESULT_CACHE		        |	 99 |
|   7 |     TABLE ACCESS FULL  | RESULT_CACHE		        |	 99 |



	ID NAME 																	               TYPE	  STATUS
----------------------------------------------------------------------------------------------------------------
	16 LWI.RESULT_CACHE																     Dependency   Published
	17  select /*+ result_cache */ c1, sum(c2) as sub_sum from result_cache group by c1	     Result	  Published
```

```sql
drop table result_cache;

create table result_cache as select mod(level, 5) c1, level c2 from dual connect by level < 10;
commit;

select c1, c2, (select /*+ result_cache*/ count(*) as scalar from result_cache) from result_cache;

select id, name, type, status from v$result_cache_objects where name like '%result_cach%' or name like '%RESULT_CACH%';

-- none

```


<br>

출처
- 오라클 성능 고도화 원리와 해법 I