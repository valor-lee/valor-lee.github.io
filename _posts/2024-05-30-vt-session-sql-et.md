---
title: '[Oracle] V$SESSION, V$SQL에서 수행시간 관련 컬럼 분석'
date: 2024-05-30 22:11:00 +09:00
categories: [database, DBMS]
tags:
  [
    DBMS,
    Oracle,
    virtual_table
  ]
---

# [Oracle] V\$SESSION, V\$SQL에서 수행시간 관련 컬럼 분석

## V$SESSION

- `LAST_CALL_ET` : 각 세션이 마지막으로 수행한 **호출 이후 경과된 시간**을 초 단위로 표시
  - 용도 : 오래 실행되고 있는 session을 정리할 수 있도록 모니터링 정보를 제공
    - session status가 active일 때 : active 시점에서 sql 실행한 시간
    - session status가 inactive일 때 : inactive 시점에서부터 지금까지 시간
- `SQL_EXEC_START` : sql_id 컬럼이 null이 아닐 때(현재 sql을 실행하고 있는 sql이 있을 때) sql을 실행한 날짜

### 오라클 테스트
두 파라미터를 기준으로 fetch 량이 많은 sql에 대해서 테스트했을 때 컬럼 값들의 추이 비교

```sql
-- session1
select sys_context('userenv', 'sessionid') from dual;
-- 7251345
drop table t1;
drop table t2;

create table t1
(c1 varchar(100),
 c2 varchar(100),
 c3 varchar(100),
 c4 varchar(100));

create table t2
(c1 varchar(100),
 c2 varchar(100),
 c3 varchar(100),
 c4 varchar(100));

insert into t1 select 1, level, level, level from dual connect by level <= 10000;
insert into t2 select 1, level, level, level from dual connect by level <= 100000;

commit;

select  *
from t1, t2
where 1=1
and t1.c1 = t2.c1;
```
```sql
-- monitoring session 
col username for a10 
set linesize 999 
SELECT sid, audsid, serial#, username, status,sql_exec_start, last_call_et FROM v$session WHERE username='usrname1' order by last_call_et desc;

SQL> SELECT sid, audsid, serial#, username, status,sql_exec_start, last_call_et FROM v$session WHERE username='usrname1' order by last_call_et desc;

       SID     AUDSID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET
---------- ---------- ---------- ---------- -------- ------------------ ------------
       756    7251344       2392 usrname1        INACTIVE                            3813
       382    3881011      14777 usrname1        KILLED                              1668
       878    7251476       5565 usrname1        INACTIVE                               8
       882    7251345      56147 usrname1        INACTIVE 30-MAY-24                     0

SQL> SELECT sid, audsid, serial#, username, status,sql_exec_start, last_call_et FROM v$session WHERE username='usrname1' order by last_call_et desc;

       SID     AUDSID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET
---------- ---------- ---------- ---------- -------- ------------------ ------------
       756    7251344       2392 usrname1        INACTIVE                            3813
       382    3881011      14777 usrname1        KILLED                              1668
       878    7251476       5565 usrname1        INACTIVE                               8
       882    7251345      56147 usrname1        ACTIVE   30-MAY-24                     0

SQL> SELECT sid, audsid, serial#, username, status,sql_exec_start, last_call_et FROM v$session WHERE username='usrname1' order by last_call_et desc;

       SID     AUDSID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET
---------- ---------- ---------- ---------- -------- ------------------ ------------
       756    7251344       2392 usrname1        INACTIVE                            3813
       382    3881011      14777 usrname1        KILLED                              1668
       878    7251476       5565 usrname1        INACTIVE                               8
       882    7251345      56147 usrname1        INACTIVE 30-MAY-24                     0
```

- 세션1에서 select 문을 수행하고 있을 때 모니터링 쿼리를 반복 수행한 결과이다.
 - audsid 7251345 row를 보면 inactive-active status로 스위칭되는 것을 확인할 수 있었다.
  - 계속 스위칭돼서 그런걸까? last_call_et가 계속 0으로 찍힌다.
    - 근데 docs에서는 active 상태면 sql 수행시간을 보여준다고 했는데 구글링해보면 0이라는 값이 active를 의미한다고도 한다.
     - inactive-active status 변환 조건이 뭐지?
      - execution할 때 active?
      - client에게 fetching 중일 때 inactive고?
      - 다시 결과값 fetch하러 특정 함수로 들어가는 순간 active가 되는 게 아닌가 싶다.
     - 수행 중이니까 sql_id가 있어서 sql_exec_start에 date가 찍혔다.

## V$SQL

아래는 두 세션에 대해서 같은 쿼리를 수행했을 때 v$sql의 elapsed time을 조회하는 쿼리이다.

```sql
select 
  b.audsid, b.sid, b.serial#, b.username, b.status, b.sql_exec_start, b.last_call_et, a.sql_id, a.elapsed_time 
from
  v$sql a, v$session b 
where 1=1 and
 a.sql_id = b.sql_id and 
 b.audsid in (7251344, 7251345) 
order by last_call_et desc;
```

퀴리 반복 수행 결과

```sql
SQL> SELECT b.audsid, b.sid, b.serial#, b.username, b.status, b.sql_exec_start, b.last_call_et, a.sql_id, a.elapsed_time FROM v$sql a, v$session b WHERE a.sql_id = b.sql_id and b.audsid in (7251344, 7251345) order by last_call_et desc;

    AUDSID        SID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET SQL_ID        ELAPSED_TIME
---------- ---------- ---------- ---------- -------- ------------------ ------------ ------------- ------------
   7251345        882      56147 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     22804178
   7251344        756       2392 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     22804170

SQL> SELECT b.audsid, b.sid, b.serial#, b.username, b.status, b.sql_exec_start, b.last_call_et, a.sql_id, a.elapsed_time FROM v$sql a, v$session b WHERE a.sql_id = b.sql_id and b.audsid in (7251344, 7251345) order by last_call_et desc;

    AUDSID        SID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET SQL_ID        ELAPSED_TIME
---------- ---------- ---------- ---------- -------- ------------------ ------------ ------------- ------------
   7251345        882      56147 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     23060146
   7251344        756       2392 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     23060138

SQL> SELECT b.audsid, b.sid, b.serial#, b.username, b.status, b.sql_exec_start, b.last_call_et, a.sql_id, a.elapsed_time FROM v$sql a, v$session b WHERE a.sql_id = b.sql_id and b.audsid in (7251344, 7251345) order by last_call_et desc;

    AUDSID        SID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET SQL_ID        ELAPSED_TIME
---------- ---------- ---------- ---------- -------- ------------------ ------------ ------------- ------------
   7251345        882      56147 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     23322971
   7251344        756       2392 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     23322955

SQL> SELECT b.audsid, b.sid, b.serial#, b.username, b.status, b.sql_exec_start, b.last_call_et, a.sql_id, a.elapsed_time FROM v$sql a, v$session b WHERE a.sql_id = b.sql_id and b.audsid in (7251344, 7251345) order by last_call_et desc;

    AUDSID        SID    SERIAL# USERNAME   STATUS   SQL_EXEC_START     LAST_CALL_ET SQL_ID        ELAPSED_TIME
---------- ---------- ---------- ---------- -------- ------------------ ------------ ------------- ------------
   7251345        882      56147 usrname1        ACTIVE   30-MAY-24                     0 0un2f5r2n1qsn     23580592
   7251344        756       2392 usrname1        INACTIVE 30-MAY-24                     0 0un2f5r2n1qsn     23580588
```


- `elapsed_time` : sql의 pp info에서 관리되는 elapsed time으로, 해당 pp가 얼마나 수행되었는지 보여주는 컬럼
 - 이 컬럼은 누적값으로 관리된다.
 - 즉, 여러 세션에서 같은 sql의 동일한 pp를 사용하게 되면 두 세션에서 수행한 시간을 elapsed_time에 누적시킨다.
