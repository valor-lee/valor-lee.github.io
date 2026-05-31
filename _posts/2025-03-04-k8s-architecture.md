---
title: '[k8s] 아키텍처(Kubernetes architecture)'
date: 2025-03-04 19:32:00 +09:00
categories: [Kubernetes, basic]
tags:
  [
    Kubernetes,
    k8s
  ]
---

# 개요

Spring Cloud의 service discovery project에 대해서 이해해보고 구현 과정을 정리하고자 합니다.

쿠버네티스(Kubernetes, K8s)는 컨테이너화된 애플리케이션을 자동으로 배포, 확장 및 운영할 수 있도록 설계된 오픈소스 오케스트레이션 플랫폼입니다.

쿠버네티스의 아키텍처는 여러 컴포넌트로 구성되며, 크게 **Control Plane(제어 플레인)**과 **Worker Node(작업자 노드)**로 나뉩니다.

# 핵심 구성 요소

![k8s architecture](https://github.com/user-attachments/assets/0b248b8c-bb6f-45ec-9f94-f56b11042d39)

- Control Plane: 클러스터를 관리하고 조정하는 역할을 수행
- Worker Node: 컨테이너를 실행하는 노드
- ETCD: 클러스터 상태 및 설정 정보를 저장하는 분산 데이터 저장소 

## 쿠버네티스의 주요 구성 요소

## 1 Control Plane (제어 플레인)

Control Plane은 클러스터의 상태를 관리하고 애플리케이션이 원활하게 실행되도록 조정하는 역할

### ① API Server (kube-apiserver)

쿠버네티스의 핵심 진입점으로, 모든 구성 요소가 이 API를 통해 상호작용

kubectl, 대시보드 등 사용자가 요청을 보낼 때 처리

RESTful API를 제공하여 클라이언트와 통신

kube-apiserver는 수평으로 확장되도록 디자인됨

### ② Controller Manager (kube-controller-manager)

여러 컨트롤러(Controller)를 실행하는 컴포넌트

주요 컨트롤러는 아래와 같다.(이외에 더 많은 컨트롤러가 존재함)

- Node Controller: 노드의 상태를 감시하고 장애 발생 시 대체 노드 할당
- ServiceAccount Controller : 신규 네임스페이스에 기본 서비스 어카운트를 생성
- Replication Controller: 파드의 개수를 유지하여 가용성을 보장
- Endpoint Controller: 서비스와 파드를 연결하여 네트워크 통신을 관리
- Job Controller : 일회성 작업 오브젝트를 감시하고 해당 작업을 수행하기 위한 파드 생성

각 컨트롤러는 논리적으로 분리된 프로세스지만, 복잡성을 낮추기 위해 단일 바이너리로 캄파일되어 단인 프로세스 내에서 실행됨


### ③ Scheduler (kube-scheduler)

클러스터 내의 워커 노드들 중 적절한 노드를 선택하여 파드를 배치

CPU, 메모리, 노드 상태 등의 리소스 사용량을 고려하여 스케줄링 수행
- **스케줄링 결정을 위해서 고려되는 요소** : 리소스에 대한 개별 및 총체적 요구 사항, 하드웨어/소프트웨어/정책적 제약, 어피니티(affinity) 및 안티-어피니티(anti-affinity) 명세, 데이터 지역성, 워크로드-간 간섭, 데드라인

### ④ etcd

모든 클러스터 데이터를 담는 쿠버네티스 뒷단의 저장소로 사용되는 일관성·고가용성 키-값 저장소.
- 클러스터의 구성 정보, 파드의 상태, 네트워크 설정
- 데이터 백업 전략 필수

Consistency(일관성)을 보장하는 Raft 알고리즘 기반

### ⑤ Cloud Controller Manager

클러스터를 클라우드 공급자의 API에 연결하고 해당 클라우드 플랫폼과 상호작용하는 컴포넌트

온프라미스 환경이나 개인 PC 학습 환경에서는 없음

- Node Controller : 노드의 응답이 멈춘 경우 클라우드에서 해당 노드가 삭제되었는지를 판단하기 위해 클라우드 공급자에 확인
- Route Controller : 클라우드 인프라스트럭처 기반에서 라우트를 설정
- Service Controller : 클라우드 공급자의 로드벨런서를 생성, 업데이트, 삭제


## 2 Worker Node (작업자 노드)

Worker Node는 실제로 컨테이너를 실행하고 관리하는 역할

### ① Kubelet

노드에서 실행되며, API Server와 통신하여 파드를 관리

파드 생성 및 라이프사이클을 제어

파드가 정상적으로 실행되고 있는지 모니터링

### ② Kube Proxy

노드 내 또는 클러스터 외부 파드로부터 네트워크 통신을 관리

서비스(서비스 디스커버리 및 로드 밸런싱) 구현부
- 운영체제에 가용한 패킷 필터링 계층을 사용
- 또는 트래픽 자체를 포워딩

iptables 또는 IPVS를 사용하여 트래픽을 파드로 전달


### ③ Container Runtime

컨테이너를 실행하는 소프트웨어 

쿠버네티스는 다양한 컨테이너 런타임 인터페이스(예: Docker, containerd, CRI-O 등)를 지원함

### ④ Pod

Kubernetes에서 배포 가능한 최소 단위

컨테이너 1~N개를 하나의 IP/네임스페이스로 묶음


## 3. 애드온

쿠버네티스 리소스를 사용하여 클러스터 기능을 제공

### ① DNS

쿠버네티스 서비스에 대한 DNS 레코드를 제공하는 DNS 서버

컨테이너들은 자동으로 DNS서버를 DNS검색에 포함함

### ② 웹 UI

클러스터를 위한 범용 웹 기반 UI

### ③ 컨테이너 리소스 모니터링

시계열 메트릭을 중앙 데이터베이스에 기록하고 탐색할 수 있는 UI

### ④ 클러스터 수준 로깅

컨테이너 로그를 검색/탐색 인터페이스를 갖춘 중앙 로그 저장소에 저장하는 역할

### ⑤ 네트워크 플러그인

컨테이너 네트워크 인터페이스 사양을 구현하는 소프트웨어 컴포넌트

파드에 IP를 할당하고 클러스터 내에서 서로 통신할 수 있도록 함


---

출처

- https://peterica.tistory.com/470
- https://kubernetes.io/ko/