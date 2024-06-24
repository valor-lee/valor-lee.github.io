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

마이크로서비스에서 각 서비스의 위치, 통신정보를 관리하여, 인스턴스 간 통신할 수 있도록 하는 기능입니다.

마이크로서비스의 운영 중에 인스턴스가 추가, 삭제 또는 정보가 변경될 수 있습니다. 그에 따라 변경되는 통신 정보를 관리해주게 됩니다.



# 설정

## Eureka server

### dependency 

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

- actuator 디펜던시는 서버 api의 통신 허용 여부를 결정하믄 데에 편리함을 제공하는 기능입니다.


### application.yml


먼저, eureka server, client project에 Eureka 모듈이 읽을 설정을 application.yml에 작성해줘야 합니다.

어떤 설정 값이 있는지는 [EurekaInstanceConfigBean.java](https://github.com/spring-cloud/spring-cloud-netflix/blob/main/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaInstanceConfigBean.java), [EurekaClientConfigBean.java](https://github.com/spring-cloud/spring-cloud-netflix/blob/main/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaClientConfigBean.java), [EurekaServerConfigBean.java](https://github.com/spring-cloud/spring-cloud-netflix/blob/main/spring-cloud-netflix-eureka-server/src/main/java/org/springframework/cloud/netflix/eureka/server/EurekaServerConfigBean.java),[common application configuration properties](https://docs.spring.io/spring-cloud-netflix/reference/configprops.html)를 참고했습니다.

너무 많은 옵션이 존재해서, 추후에 discovery server 기능을 고도화할 때 자세히 알아보도록 하겠습니다.

EurekaClientConfigBean.class에서는 아래와 같이 client configuration에 아래와 같이 필수 기본값들을 설정되는 것으로 확인됩니다.

```java
...

@ConfigurationProperties("eureka.client")
public class EurekaClientConfigBean implements EurekaClientConfig, Ordered {

    ...

    public static final String PREFIX = "eureka.client";
    public static final String DEFAULT_URL = "http://localhost:8761/eureka/";
    public static final String DEFAULT_ZONE = "defaultZone";
    private static final int MINUTES = 60;
    private boolean enabled = true;
    private int registryFetchIntervalSeconds = 30;
    private int instanceInfoReplicationIntervalSeconds = 30;
    private int initialInstanceInfoReplicationIntervalSeconds = 40;
    private int eurekaServiceUrlPollIntervalSeconds = 300;
    private int eurekaServerReadTimeoutSeconds = 8;
    private int eurekaServerConnectTimeoutSeconds = 5;
    private int eurekaServerTotalConnections = 200;
    private int eurekaServerTotalConnectionsPerHost = 50;
    private String region = "us-east-1";
    private int eurekaConnectionIdleTimeoutSeconds = 30;
    private int heartbeatExecutorThreadPoolSize = 2;
    private int heartbeatExecutorExponentialBackOffBound = 10;
    private int cacheRefreshExecutorThreadPoolSize = 2;
    private int cacheRefreshExecutorExponentialBackOffBound = 10;
    private Map<String, String> serviceUrl = new HashMap();
    public EurekaClientConfigBean() {
      this.serviceUrl.put("defaultZone", "http://localhost:8761/eureka/");
      this.gZipContent = true;
      this.useDnsForFetchingServiceUrls = false;
      this.registerWithEureka = true;
      this.preferSameZoneEureka = true;
      this.availabilityZones = new HashMap();
      this.filterOnlyUpInstances = true;
      this.fetchRegistry = true;
      this.dollarReplacement = "_-";
      this.escapeCharReplacement = "__";
      this.allowRedirects = false;
      this.onDemandUpdateStatusChange = true;
      this.clientDataAccept = EurekaAccept.full.name();
      this.shouldUnregisterOnShutdown = true;
      this.shouldEnforceRegistrationAtInit = false;
      this.order = 0;
    }
    ...
}

```


EurekaInstanceConfigBean.class에서는 아래와 같이 server configuration에 아래와 같이 필수 기본값들을 설정되는 것으로 확인됩니다.


```java
@ConfigurationProperties("eureka.instance")
public class EurekaInstanceConfigBean implements CloudEurekaInstanceConfig, EnvironmentAware {
    private static final String UNKNOWN = "unknown";
    private String actuatorPrefix = "/actuator";
    private String appname = "unknown";
    private int nonSecurePort = 80;
    private int securePort = 443;
    private boolean nonSecurePortEnabled = true;
    private int leaseRenewalIntervalInSeconds = 30;
    private int leaseExpirationDurationInSeconds = 90;
    private String virtualHostName = "unknown";
    private String secureVirtualHostName = "unknown";
    private Map<String, String> metadataMap = new HashMap();
    private EurekaInstanceConfigBean() {
        this.dataCenterInfo = new MyDataCenterInfo(Name.MyOwn);
        this.statusPageUrlPath = this.actuatorPrefix + "/info";
        this.homePageUrlPath = "/";
        this.healthCheckUrlPath = this.actuatorPrefix + "/health";
        this.namespace = "eureka";
        this.preferIpAddress = false;
        this.initialStatus = InstanceStatus.UP;
        this.defaultAddressResolutionOrder = new String[0];
    }
}
```

필수적인 옵션은  default로 설정되어있고, Eureka Server 옵션 몇 가지를 아래와 같이 설정했습니다.

```yml
server:
  port: 8761

spring:
  application:
    name: service-discovery

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    # registry를 갱신할 수 없을 때 대기하는 시간
    wait-time-in-ms-when-sync-empty: 1000

management:
  endpoints:
    web:
      exposure:
        include: "*"

```
- `register-with-eureka` : service discovery에 본인의 정보를 등록할지 여부를 결정합니다.
- `fetch-registry` : service discovery에서 등록된 서비스들의 정보를 가져와 현재 서버 내에 캐싱할지 여부를 결정합니다.
  - server에서는 service discovery instance가 다수 일 때 true로 켜주게 됩니다.
- `wait-time-in-ms-when-sync-empty` : service들의 정보를 가져올 수 없을 때 기다릴 시간을 설정합니다. 
- `management.endpoints.web.exposure.include` : 외부 접근에서 허용하는 api를 설정하는 Spring actuator 관련 옵션입니다.

출처

[Spring Cloud Nexflix dosc 1](https://cloud.spring.io/spring-cloud-netflix/multi/multi__service_discovery_eureka_clients.html)