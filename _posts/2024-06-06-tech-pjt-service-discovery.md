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

마이크로서비스에서 각 서비스의 통신정보를 관리하여, 인스턴스 간 통신할 수 있도록 하는 기능입니다.

마이크로서비스의 운영 중에 인스턴스가 추가, 삭제 또는 정보가 변경될 수 있습니다. 그에 따라 변경되는 통신 정보를 관리해주게 됩니다.

# 프로젝트 구성

```
├── README.md
├── order
│   ├── build
│   ├── build.gradle
│   ├── gradle
│   ├── gradlew
│   ├── gradlew.bat
│   ├── settings.gradle
│   └── src
│       ├── main
│       │   ├── java
│       │   └── resources
│       └── test
├── product
│   ├── build
│   ├── build.gradle
│   ├── gradle
│   ├── gradlew
│   ├── gradlew.bat
│   ├── settings.gradle
│   └── src
│       ├── main
│       │   ├── java
│       │   └── resources
│       └── test
└── service-discovery
    ├── build
    ├── build.gradle
    ├── gradle
    ├── gradlew
    ├── gradlew.bat
    ├── settings.gradle
    └── src
        ├── main
        │   ├── java
        │   └── resources
        └── test
```
위와 같이 order, product 프로젝트와 service-discovery 프로젝트를 구성했습니다.


# 설정

## 1. Eureka server

### dependency 

Spring boot 3.3.0으로 시작하여 [Spring boot compatibility table](https://github.com/spring-cloud/spring-cloud-release/wiki/Supported-Versions#supported-releases)에 따라 4.1.x 이상인 4.1.2를 추가했습니다.


```groovy
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

필수적인 옵션은 default로 설정되어있고, Eureka Server 옵션 몇 가지를 아래와 같이 설정했습니다.

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
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  
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

### Eureka Server annotation

```java
...

import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class ServiceDiscoveryApplication {

	public static void main(String[] args) {
		SpringApplication.run(ServiceDiscoveryApplication.class, args);
	}
}
```

- cloud.netflix.eureka.server package에서 EnableEurekaServer 어노테이션을 붙입니다.




## 2. Eureka Client(Order/Product Service)

Eureka Client로서 아래와 같이 설정해줬습니다.


### dependency 

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.3.0'
	id 'io.spring.dependency-management' version '1.1.5'
}

version = '0.0.1-SNAPSHOT'

java {
	sourceCompatibility = '21'
}

dependencies {
	implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:4.1.2'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'

  ...
}

```

### application.yml

```yaml
server:
  port: 8001

spring:
  application:
    name: product

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    disable-delta: true
    eureka-connection-idle-timeout-seconds: 30
    eureka-server-connect-timeout-seconds: 5
    eureka-server-read-timeout-seconds: 8
    eureka-server-total-connections: 200
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```
- `register-with-eureka` : service discovery에 본인의 정보를 등록할지 여부를 결정합니다.
- `fetch-registry` : service discovery에서 등록된 서비스들의 정보를 가져와 현재 서버 내에 캐싱할지 여부를 결정합니다.
- `disable-delta` : service discovery에서 정보를 가져올 때 변경된 정보만 가져올 지 여부에 대한 설정입니다.
  - 통신 대역폭 자원을 절약할 수 있는 옵션입니다.
- `wait-time-in-ms-when-sync-empty` : service들의 정보를 가져올 수 없을 때 기다릴 시간을 설정합니다. 
- `management.endpoints.web.exposure.include` : 외부 접근에서 허용하는 api를 설정하는 Spring actuator 관련 옵션입니다.


### Eureka Client annotation

```java
...
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@EnableDiscoveryClient
@SpringBootApplication
public class OrderApplication {

	public static void main(String[] args) {
		SpringApplication.run(OrderApplication.class, args);
	}

}

```


- spring-cloud-commons 라이브러리의 EnableDiscoveryClient 어노테이션을 추가합니다.
  - spring-cloud-netflix 라이브러리의 EnableEurekaClient도 있지만, Eureka 관련 기능만 의존하고 있고 기능이 제한적입니다.


위와 같이 개발이 완료되면 http://localhost/8761/ 로 아래와 같이 Eureka Server 대시보드를 확인할 수 있습니다.

![image](https://github.com/valor-lee/valor-lee.github.io/assets/109330610/43ca7e97-70d7-462a-872b-6edbb673e7d3)

Instances currently registered with Eureka : 유레카에 등록된 클라이언트 목록을 볼 수 있습니다.
1. application : application.yml에서 설정한 spring.application.name에 설정된 서비스 명을 나타냅니다.
2. AMIs : 아직 정확한 의미를 파악하지 못했습니다....
3. Availability Zones : 각 서비스의 인스턴스 개수를 의미합니다.
4. Status : 각 서비스의 등록 상태 및 Url 정보를 표기합니다.


# Eureka API

> 참고
> [Eureka REST operations 문서 링크](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations) 에서 Eureka Server가 서비스 관리를 위해 사용하는 API를 확인할 수 있습니다.

이제 테스트 목적으로 각 서버에서 Eureka Server에 등록된 통신정보를 조회하는 API를 개발했습니다.

```java
@RestController
public class DiscoveryController {
    private final DiscoveryService discoveryService;

    public DiscoveryController(DiscoveryService discoveryService) {
        this.discoveryService = discoveryService;
    }

    @GetMapping("/registry")
    public List<String> getServices() {
        return discoveryService.getServices();
    }
}

---
...
import org.springframework.cloud.client.discovery.DiscoveryClient;

@Service
@RequiredArgsConstructor
public class DiscoveryService {

    private final DiscoveryClient discoveryClient;

    public List getServices() {
        List<String> services = new ArrayList<>();

        discoveryClient.getServices()
                .forEach(serviceId -> {discoveryClient.getInstances(serviceId)
                        .forEach(serviceInstance ->
                                services.add(String.format("%s : %s", serviceId, serviceInstance.getUri())));
                });
        return services;
    }
}
```

- Controller와 Service로 분리하여 통신 정보를 조회합니다.
- spring-cloud-commons 라이브러리에서 제공하는 `DiscoveryClient` 객체를 활용하여 Service Discovery에서 가져온 instance 정보들을 가지옵니다.


<img src="https://github.com/valor-lee/valor-lee.github.io/assets/109330610/7a98763c-b8d1-4f1c-8e82-c1efc91344c2" width="400px" height="150px" title="Github_Logo"></img>
<img src="https://github.com/valor-lee/valor-lee.github.io/assets/109330610/b09048bf-c8cd-44c5-9ccf-a2910d74e336" width="400px" height="150px" title="Github_Logo"></img>

- 이제 각 프로젝트를 실행하여 postman으로 테스트해보면 위와 같이 각 instance정보가 잘 조회됨을 확인할 수 있습니다.


---

이상으로, Service Discovery 개발은 끝으로 하며, 추후에  고도화 작업을 진행하고자 합니다.




---

출처

- [Spring Cloud Nexflix dosc 1](https://cloud.spring.io/spring-cloud-netflix/multi/multi__service_discovery_eureka_clients.html)
- [5. [MSA 구현 퀵스타트] 서비스 디스커버리 초간단 구현](https://enjoy-dev.tistory.com/2?category=936724)
- [coe github blog](https://coe.gitbook.io/guide/service-discovery/eureka)
- https://authentication.tistory.com/24
- https://wellbell.tistory.com/230
- https://ssipflow.github.io/msa/Spring-Cloud-API-Gateway-04/
- https://adjh54.tistory.com/207#3)%20Spring%20Cloud%20%EC%A2%85%EB%A5%98%20%3A%20Component-1