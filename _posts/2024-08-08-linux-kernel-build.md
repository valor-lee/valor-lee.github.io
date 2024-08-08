---
title: Linux kernel build
date: 2024-08-08 20:29:00 +09:00
categories: [Linux]
tags:
  [
    Linux,
    kernel
  ]
---


# 단계별 Linux Kernel Build

## 1. Dockerfile 작성

```Dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive
ARG USERNAME=user1

RUN apt-get update && \
                apt-get upgrade && \
                apt-get install -y sudo

RUN     apt-get install -y git bc bison flex libssl-dev make pkg-config ncurses && \
                apt-get install -y vim

# add user valor
RUN adduser --disabled-password --gecos '' $USERNAME && \
                usermod -aG sudo $USERNAME && \
                echo "$USERNAME  ALL=(ALL)  NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME && \
                chmod 0440 /etc/sudoers.d/$USERNAME

USER $USERNAME

WORKDIR /home/$USERNAME

RUN git clone --depth=1 https://github.com/torvalds/linux.git
```
- 필수 툴들을 설치하고 linux git repository를 다운로드합니다.

## 2. Docker 컨테이너 실행 및 접속

```bash
docker build -t linux-kernel .
docker run -it --name linux-kernel -v $(pwd):/home/user1 linux-kernel
docker exec -it -u user1 linux-kernel bash
```

## 3. 커널 설정

```bash
# 기본 설정 사용
make defconfig

# 터미널 gui를 사용한 설정
make menuconfig
```
- 커널을 빌드하기 전에 설정을 구성합니다.
- 기본설정을 사용하거나 사용자 정의 설정을 할 수 있습니다.
- .config파일이 생깁니다.


## 4. 커널 컴파일

```bash
make -j$(nproc)
```
- nproc : 시용 가능한 모든 CPU 코어 수 만큼의 병렬 빌드하게 됩니다.
- 이 명령어로 커널과 모듈을 빌드하데 됩니다.
  - 모듈은 커널 기능을 확장하기 위해 개별정인 드라이버 또는 기능을 의미합니다.
  - 모듈은 `*.ko`(Kernel Object)파일로 컴파일됩니다.

## 5. 커널 설치
- docker에서 테스트만 할 것이라 저는 skip했습니다.
```bash
# root 권한 실행
sudo make modules_install

# 일반사용자로 실행
make INSTALL_MOD_PATH=/workspace/modules modules_install
```
- 빌드된 모듈들을 시스템의 표준 위치에 복사됩니다.
  - 시스템 표준 위치란 `/lib/modules/<커널버전>/` 디렉토리를 생성해서 복사합니다.
- 일반 유저의 경우 특정 경로에 설치할 수 있습니다. Docker 컨테이너와 같은 격리 환경에서 커널 모듈을 빌드하고 관리하는 경우 유용합니다.
- 



---

출처