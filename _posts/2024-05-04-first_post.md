---
title: Nested loop Join
date: 2024-05-04 17:56:00 +09:00
categories: [database, DBMS]
tags:
  [
    DBMS,
    join
  ]
  
---

# Nested Loop Join

## 목차

1. 메커니즘
2. 실행 예시
3. 튜닝
  1. prefetch
  2. batch IO
  3. buffer pinning


## 1. 메커니즘

조인하는 두 테이블에 대해서, 

outer table, inner table을 정한 후

outer table에서 scan 해온 블럭 내에 row들로 조인키 조건에 만족하는 inner table 내에 row들과 결합하게 됩니다.

## 2. 실행 예시

