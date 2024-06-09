---
title: MSA기반 백엔드 기술 프로젝트 (1. Service Dsicovery)
date: 2024-06-06 19:32:00 +09:00
categories: [Framework, Spring, Spring Cloud]
tags:
  [
    project,
    backend,
    Spring, Spring Cloud,
    service discovery,
    MSA
  ]
---

# 개요

Spring Cloud의 service discovery project에 대해서 이해해보고 구현 과정을 정리하고자 합니다.

# Service Discovery란

![image](https://github.com/valor-lee/valor-lee.github.io/assets/109330610/1320b2f3-1ad3-439b-84f2-9be514d395c9)

[출처](https://www.springcloud.io/post/2022-03/spring-cloud-introduction-to-service-discovery-netflix-eureka/#gsc.tab=0)

마이크로서비스에서 각 서비스의 위치, 통신정보를 관리하여 통신할 수 있도록 하는 기능입니다.

마이크로서비스의 운영 중에 인스턴스가 추가, 삭제 또는 정보가 변경될 수 있습니다. 그 간에 변경되는 통신 정보를 관리해주게 됩니다.



# 설정

## dependency 

Spring boot 3.3.0으로 시작하여 [Spring boot compatibility table](https://github.com/spring-cloud/spring-cloud-release/wiki/Supported-Versions#supported-releases)에 따라 4.1.x 이상인 4.1.2를 추가했습니다.


```gradle
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.3.0'
	id 'io.spring.dependency-management' version '1.1.5'
}

java {
	sourceCompatibility = '21'
}

...

dependencies {
	implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server:4.1.2'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
  ...
}
```

## application.yml


먼저, eureka server, client project에 Eureka 모듈이 읽을 설정을 application.yml에 작성해줘야 합니다.

어떤 설정 값이 있는지는 [EurekaInstanceConfigBean.class](https://github.com/spring-cloud/spring-cloud-netflix/blob/main/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaInstanceConfigBean.java), [EurekaClientConfigBean.class](https://github.com/spring-cloud/spring-cloud-netflix/blob/main/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaClientConfigBean.java), [common application configuration properties](https://docs.spring.io/spring-cloud-netflix/reference/configprops.html)를 참고했습니다.

너무 많은 옵션이 존재해서, 추후에 discovery server 기능을 고도화할 때 자세히 알아보도록 하겠습니다.

일단 기동을 위한 기본적인 설정을 아래와 같이 추가했습니다.



출처

[Spring Cloud Nexflix dosc 1](https://cloud.spring.io/spring-cloud-netflix/multi/multi__service_discovery_eureka_clients.html)