---
title: Linux kernel 분석하기
date: 2024-08-08 11:29:00 +09:00
categories: [Linux]
tags:
  [
    Linux,
    kernel
  ]
---


# 리눅스 커널 분석하기

## vmlinux

- 컴파일된 후의 원본 리눅스 커널의 이미지
- 디버깅 정보와 기타메타데이터를 포함할 수 있기 때문에 파일 크기가 큼

## vmlinuz(또는 bzImage)
- vmlinux파일로 만든 v86 용 압축 파일
- 부팅과정에서 이 압축파일을 풀어 vmlinux와 같은 원래 커널 이미지를 메모리에 로드함

## kernel8.img
- vmlinux파일로 만든 Armv8 용 압축 파일


# Makefile

## KBUILD_CFLAGS

```Makefile
KBUILD_CFLAGS += -std=gnu11
KBUILD_CFLAGS += -fshort-wchar
KBUILD_CFLAGS += -funsigned-char
KBUILD_CFLAGS += -fno-common
KBUILD_CFLAGS += -fno-PIE
KBUILD_CFLAGS += -fno-strict-aliasing
KBUILD_CFLAGS += -fno-delete-null-pointer-checks
KBUILD_CFLAGS += -O2
KBUILD_CFLAGS += -Os
...
```

gcc로 c파일을 컴파일 할 때 gcc 옵션을 생성해줍니다.

## gcc 옵션 알아보기

### `-save-temps=obj`

- 전처리가 끝난 소스코드를 볼 수 있도록 해주는 gcc 옵션입니다.
- *.i, *.s 파일이 떨어지게 됩니다.
  - *.i : 전처리된 파일
  - *.s : 컴파일러가 컴파일한 어셈블리 코드 파일()

### `-W`

- compile 과정에서 경고메세지 수준 및 종류들을 설정하는 옵션입니다.
  - `-Wall` : 모든 모호한 코딩에 대해 경고 
  - `-W` : 합법적이지만 모호한 코딩에 대해 경고를 보내는 옵션
  -  `-W -Wall` : 아주 사소한 모호성에 대해서도 경고가 발생

### `-O2`
- 머신에 종속적인 최적화 기법을 레벨 2로 설정합니다.
- 사이즈와 성능에 트레이드 오프가 없는 많은 최적화를 수행하게 됩니다.

### `-l`

- 라이브러리들을 참조해서 넣을 때 사용하는 옵션입니다.

### `-std=gnu11`

- 2011년도에 출시된 gcc 표준에 따라 문법을 허용하여 컴파일하도록 설정하는 옵션입니다.

### `-m`

- 아키텍처 종속 옵션을 설정하는 옵션입니다.

## `-f`

- 컴파일러의 특정 기능을 끄고 켤 때 사용하는 옵션입니다.
- 보통 최적화, 컴파일러 자체의 디버깅, 문법 체크 등과 관련있습니다.

