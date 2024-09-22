---
title: Linux 역사
date: 2024-07-14 10:29:00 +09:00
categories: [Linux]
tags:
  [
    Linux,
    kernel
  ]
---


# 리눅스의 역사

![image](https://github.com/user-attachments/assets/41896aac-b4ed-4f80-8562-97153976a0da)

컴퓨터가 개발된 이후 한 동안 한가지 프로그램만 실행할 수 있었습니다.

## UNIX 탄생과 성장

### 1960년 대 후반 : Multics 프로젝트, UNIX 탄생 

MIT, AT&T 벨 연구소, General Electric에서  Multics 운영체제 개발 진행했습니다.

> Multics : 멀티태스킹, 멀티유저를 지원하는 초기 형태의 시분할 운영체제

초기 설계 목표와 달리 비대해지고 쓸모 없는 개발이 되어 가다가 프로젝트가 실패하게 됩니다.

이후 일부 연구원들이 모여 멀티 캐스팅 + 멀티 유저 지원 운영체제인 Unix를 개발하게 됩니다.

하지만 문제는 당시 Unix가 어셈블리어로 개발되었기 때문에, CPU에 따라 운영체제 이식을 위해 CPU별 어셈블리어로 추가 개발이 요구되었습니다.


### 1973년 : C언어 탄생과 UNIX의 재탄생

이러한 상황에서 데니스 리치(Dennis Richie)가 C언어를 개발하게 됩니다.

Unix 프로그램을 C언어로 재작성하여 이식성과 호환성이 높은 Unix로 재탄생하게 됩니다.

이후 오픈소스로 관리되던 Unix는 수많은 인력과 함께 발전하게 되며 완성도를 높아지게 되었습니다.

### 1984년 : GNU의 등장과 Unix의 발전

당시에 소위 인기가 많고 영향력이 높았던 AT&T가 정부로부터의 반독점 소송에 패배하며 분사 및 컴퓨터 사업에 진출하지 못하게 됩니다.

이로 인해 유닉스를 개발할 필요가 없어지면서 유닉스를 유료화하여 판매합니다.

이 과정에서 구매한 기업은 Unix 소스코드를 수정할 수도 있게 되어 변종들이 탄생하게 되어 혼란이 발생되었습니다.

이 문제를 해결하기 위해 유닉스에 대한 표준인 `POSIX 규격`이 만들어 유닉스 벤더들이 호환성을 유지한 제품을 개발하게 됩니다.

기존에 유닉스를 개발했던 대학 및 연구기관에서 유닉스를 무료로 소스코드를 볼 수 없게 되자, 무료버전 유닉스를 개발하자는 목표의 **GNU(GNU is not Unix)** 단체를 설립합니다.

이 시기에 GNU는 FSF(Free Software Foundation)을 설립하여 FSF를 중심으로 수익활동을 통해 GNU 프로젝트 진행 비용을 벌게 됩니다.

하지만 수년간, 운영체제의 핵심인 커널(kernel) 개발에 난항을 겪게 됩니다.

## LINUX 탄생과 성장

### 1991년 리누스 토발즈의 등장과 Linux 탄생

필란드 헬싱키의 21살의 대학생이 인텔 i387에서 운영되는 유닉스 호환운영체제를 개발합니다.

초기버전 0.01은 기본적인 커널 기능, 0.02 버전은bash(GNU Bounced Again Shell)와 gcc(GNU C 컴파일러)가 실행, 0.95 버전에서 인텔 x86에서 그래픽 사용자 인터페이스 기능까지 지원하게 되었습니다.

당시 Hurd라는 커널을 개발하던 FSF는 커널 개발을 포기하고 리누스 토발즈가 개발한 커널은 GNU 커널로 공식 채택하게 되며, Linux가 탄생하게 됩니다.

### 1994년 : 리눅스 1.0 배포

네트워킹 기능이 추가되어 리눅스 1.0이 배포되었습니다.
- 이후 이를 수익모델로 하여 래드햇 사가 출범하게 됩니다.

### 1995년 

텔, 디지털, 썬 스팍 프로세스에도 포팅됨으로 그 영역을 넓혔으며, 알파프로세서용의 64비트 리눅스도 등장했습니다.

### 1996년 

여러 프로세서를 한 번에 사용할 수 있는 컴퓨팅 파워가 추가된 버전 2.0 출시됩니다.

### 1999년

SMP(Symmentric Multiprocessor System) 기능을 지원하게 됩니다.
- 최대 16개의 CPU 장착 가능
- 최대 동시 접속자 2048 명 지원


> SMP(Symmentric Multiprocessor System)
>
> 두 개 이상의 동일한 프로세서가 하나의 메모리, I/O 디바이스, 인터럽트 등의 자원을 공유하여 단일 시스템 버스를 통해, <br>각 프로세서가 다른 프로그램을 실행하고 다른데이터를 처리하는 시스템을 의미합니다.
> 
> [상세 정보](https://myfreechild.tistory.com/entry/SMPSymmetric-MultiProcessing-vs-AMPAsymmetric-MultiProcessing)
---

출처
- https://www.youtube.com/watch?v=afkwBG39vzk&list=PLz--ENLG_8TPuiK-Ib4uW5DXPvsdCDNc1&index=2
- https://12bme.tistory.com/220
- https://blog.naver.com/crushhh/221564455731