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

쿼리나 pl/sql 수행 결과를 shared memory에 캐싱해두고, 동일한 쿼리가 수행될 때 block IO나 연산 없이 결과값을 반환해주는 기능이다.

## 2. 특징


## 3. 메커니즘

1. result_cache_mode를 켜주고, 결과를 캐싱하고 싶은 sql에 `RESULT_CACHE` 힌트를 작성한다.
2. 서버에서 요청에 대해서 SGA의 shared pool의 result cache 메모리에서 찾아본다.
   1. hit : 





## 3. 실행 예시

## 4. 튜닝




출처

오라클 성능 고도화 원리와 해법 I

https://positivemh.tistory.com/1026