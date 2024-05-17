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
  - [3. 실행 예시](#3-실행-예시)
  - [4. 튜닝](#4-튜닝)


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


**관련 파라미터**
- `result_cache_mode`(manual(default)/force)
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


## 3. 실행 예시

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

```sql
SQL> select /*+result_cache*/ * from all_tables;

...
2218 rows selected.


SQL> select name, scan_count from v$result_cache_objects where name like '%all_tables%';

no rows selected

```

## 4. 튜닝




출처

오라클 성능 고도화 원리와 해법 I

https://positivemh.tistory.com/1026