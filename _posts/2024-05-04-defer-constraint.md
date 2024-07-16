---
title: '[Oracle] Constraint Deferred'
date: 2024-07-16 19:59:00 +09:00
categories: [database, DBMS]
tags:
  [
    DBMS,
    Oracle,
    constraint
  ]
  
---

# [Oracle] Constraint Deferred

기본적으로 dml 수행할 떄 row by row로 각 constraint를 검증하면서 위반했을 경우 statement rollback이 발생합니다.

Constraint에 Defer 옵션을 걸게 되면, transaction 단위로 제약조건 위반 여부에 따라 tx rollback 실행하게 됩니다.


## Constraint Deferred를 사용하는 이유

복잡한 참조 관계에 있어서 트랜잭션을 처리해야 할 때 유용합니다.

예를 들어, 

```sql
-- 부모 테이블 생성
CREATE TABLE parent_table (
    id NUMBER PRIMARY KEY,
    name VARCHAR2(50)
);

-- 자식 테이블 생성
CREATE TABLE child_table (
    id NUMBER PRIMARY KEY,
    parent_id NUMBER,
    CONSTRAINT fk_parent
        FOREIGN KEY (parent_id)
        REFERENCES parent_table(id)
);
```
위와 같이, parent_table의 기본키인 id를 참조하고 있는 child_table의 외래키인 parent_id를 업데이트해야 할 때나 id를 업데이트헤야 하는 경우,<br> <span style='color:red'>constraint violation이 발생</span>하게 됩니다.


**constraint deferred** 기능을 사용하면 tx가 완료되었을 때 위반여부를 판단하기 때문에<br>
트랙잭션 내에서 잠시 데이터 무결성이 깨지더라도 commit 시점에서 제약 조건 위배 여부를 판단하기 때문에 편리하게 트랜잭션을 완료할 수 있습니다.





## 사용법

### deferrable | not deferrable 

constraint를 생성할 때 deferrable 여부를 결정해야 합니다.
- 즉, not deferrable인 constraint는 deferred 설정할 수 없다.
- default : not deferrable

```sql
SQL> alter table t 
      add constraint con1 check(col > 10)
      deferrable;

SQL> select status, deferrable, deferred from user_constraints where constraint_name = 'con1';

```

### deferred | immediate

default 값은 immediate 입니다.

deferrable constraint에 대해서 아래 문법을 통해 deferred 상태로 변경해주면 됩니다.


### 1. constraint 생성 시 deffered 문법

```sql
-- 부모 테이블 생성
CREATE TABLE parent_table (
    id NUMBER PRIMARY KEY,
    name VARCHAR2(50),
    CONSTRAINT chk_name CHECK (name > 0) DEFERRABLE INITIALLY DEFERRED

);

-- 자식 테이블 생성
CREATE TABLE child_table (
    id NUMBER PRIMARY KEY,
    parent_id NUMBER,
    CONSTRAINT fk_parent
        FOREIGN KEY (parent_id)
        REFERENCES parent_table(id)
        DEFERRABLE INITIALLY DEFERRED
);

```

-  `DEFERRABLE INITIALLY DEFERRED` : 트랜잭션이 끝날 때까지 제약 조건 검사를 지연시킵니다.



### 2. 기존 constraint를 deferred 문법


```sql
-- 기존 테이블에 대한 제약 조건을 지연 평가로 변경
ALTER TABLE child_table
MODIFY CONSTRAINT chk_name DEFERRABLE INITIALLY DEFERRED;

ALTER TABLE child_table
MODIFY CONSTRAINT fk_parent DEFERRABLE INITIALLY DEFERRED;
```

### 3. 트랜잭션 내에서 deferred 문법

```sql
-- 트랜잭션 내에서 모든 제약 조건을 지연 평가로 설정
SET CONSTRAINTS ALL DEFERRED;

-- 특정 제약 조건을 지연 평가로 설정
SET CONSTRAINTS chk_name DEFERRED;
SET CONSTRAINTS fk_parent DEFERRED;

-- 모든 제약 조건을 즉시 평가로 설정
SET CONSTRAINTS ALL IMMEDIATE;

-- 특정 제약 조건을 즉시 평가로 설정
SET CONSTRAINTS chk_name IMMEDIATE;
SET CONSTRAINTS fk_parent IMMEDIATE;
```


