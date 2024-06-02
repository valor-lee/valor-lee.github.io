---
title: '[Oracle] V$SESSION, V$SQL���� ����ð� ���� �÷� �м�'
date: 2024-05-30 22:11:00 +09:00
categories: [database, DBMS]
tags:
  [
    DBMS,
    Oracle,
    virtual_table
  ]
---

# [Oracle] V\$SESSION, V\$SQL���� ����ð� ���� �÷� �м�

## V$SESSION

- `LAST_CALL_ET` : �� ������ ���������� ������ **ȣ�� ���� ����� �ð�**�� �� ������ ǥ��
  - �뵵 : ���� ����ǰ� �ִ� session�� ������ �� �ֵ��� ����͸� ������ ����
    - session status�� active�� �� : active �������� sql ������ �ð�
    - session status�� inactive�� �� : inactive ������������ ���ݱ��� �ð�
- `SQL_EXEC_START` : sql_id �÷��� null�� �ƴ� ��(���� sql�� �����ϰ� �ִ� sql�� ���� ��) sql�� ������ ��¥

### ����Ŭ �׽�Ʈ
�� �Ķ���͸� �������� fetch ���� ���� sql�� ���ؼ� �׽�Ʈ���� �� �÷� ������ ���� ��

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

- ����1���� select ���� �����ϰ� ���� �� ����͸� ������ �ݺ� ������ ����̴�.
 - audsid 7251345 row�� ���� inactive-active status�� ����Ī�Ǵ� ���� Ȯ���� �� �־���.
  - ��� ����Ī�ż� �׷��ɱ�? last_call_et�� ��� 0���� ������.
    - �ٵ� docs������ active ���¸� sql ����ð��� �����شٰ� �ߴµ� ���۸��غ��� 0�̶�� ���� active�� �ǹ��Ѵٰ� �Ѵ�.
     - inactive-active status ��ȯ ������ ����?
      - execution�� �� active?
      - client���� fetching ���� �� inactive��?
      - �ٽ� ����� fetch�Ϸ� Ư�� �Լ��� ���� ���� active�� �Ǵ� �� �ƴѰ� �ʹ�.
     - ���� ���̴ϱ� sql_id�� �־ sql_exec_start�� date�� ������.

## V$SQL

�Ʒ��� �� ���ǿ� ���ؼ� ���� ������ �������� �� v$sql�� elapsed time�� ��ȸ�ϴ� �����̴�.

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

���� �ݺ� ���� ���

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


- `elapsed_time` : sql�� pp info���� �����Ǵ� elapsed time����, �ش� pp�� �󸶳� ����Ǿ����� �����ִ� �÷�
 - �� �÷��� ���������� �����ȴ�.
 - ��, ���� ���ǿ��� ���� sql�� ������ pp�� ����ϰ� �Ǹ� �� ���ǿ��� ������ �ð��� elapsed_time�� ������Ų��.
