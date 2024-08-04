---
title: Java Spring HTTP 통신 방식
date: 2024-07-12 00:17:00 +09:00
categories: [Framework, Spring]
tags:
  [
    backend,
    Spring,
    http
  ]
---

# 개요

Spring Framework에서 주로 사용하는 HTTP 요청 방식에 대해서 공부한 내용을 정리하고자 합니다.

1. RestTemplate
2. WebClient
3. OpenFeign

순으로 정리하겠습니다.

# RestTemplate

- Spring Framework에서 제공하는 **Multi Thread & Synchronous blocking IO HTTP 통신**을 위한 클라이언트
- HTTP  메서드 지원 및 JSON, XML 등 다양한 형식의 데이터를 처리 가능
- deprecated 될 예정
  
![image](https://github.com/user-attachments/assets/4df90c76-1e21-45cd-856e-35ffd5eeea09)
- 어플리케이션 구동시 미리 Thread pool을 확보합니다.
- Client 요청은 Queue에 쌓이고 가용한 스레드에 할당되어 처리합니다.
- 각 스레드는 응답이 올 때까지 blocking됩니다.


## RestTemplate 단점
- 가용한 스레드가 없을 경우 요청이 queue에서 대기하게 된다.
- 대부분의 문제는 네트워킹이나 DB와의 통신 문제가 생길 수 있음
- 락이나 병목현상에 의한 서비스 느려지는 문제 발생할 수 있음

# WebClient
```java

```
- Spring Framework 5 이상에서 제공
- **Single Thread & Asynchronous & Nonblocking IO HTTP 통신**을 위한 클라이언트

![image](https://github.com/user-attachments/assets/da0bb7bf-6ad2-43ec-bf58-7c60c4d0edd0)

- 각 요청이 Event Loop 내에 Job으로 등록됩니다.
- Event Loop는 각 Job을 provider에게 요청 후, 다른 Job을 처리합니다.
- provider에게 callback이 오면 그 결과를 요청자에게 제공합니다.
- 위 기능을 위해 반응성, 탄력성, 가용성, 비동기성을 보장하는 Spring React 프레임워크를 사용합니다.
- 또한, React Web 프레임워크인 Spring WebFlux에서 HTTP Client를 사용합니다.

## 장점

- Nonblocking 방식으로 네트워크 병목 현상을 줄이고 성능을 향상시킬 수 있습니다.
  - 이를 통한 consumer-provider 사이 통신을 더 효율적으로 진행할 수 있습니다.

## 사용법

![image](https://github.com/user-attachments/assets/5b9cfa19-6266-4a74-8f2d-258c3d7ecd49)

1. WebClient Builder로 원하는 옵션 작성
  - 연결할 주소와 포트, HTTP Method, Header 등 설정

2. Builder 기반 Publisher 생성
  - response 받을 값의 형태를 정의
  - 단일값을 받을 건지(mono), 여러 값을 받을 건지(flux) 설정
  - body 설정(bodyToMono, bodyToFlux)
  
3. 실제 통신을 시도하여 값을 받는다.
  - publisher 객체(mono | flux)에 Subscribe 함수를 사용하거나 stream에 collect 함수를 사용해서도 값을 받을 수 있음
  - publisher 객체에 block 함수를 이용하여 동기식으로 값을 받을 수 있지만 권장되지 않음



---

출처

- https://hahahoho5915.tistory.com/79
- https://happycloud-lee.tistory.com/220