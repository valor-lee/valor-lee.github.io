---
title: 'What Every Programmer Should Know About Memory - Chapter6.1 Bypassing the Cache'
date: 2026-01-02 19:32:00 +09:00
categories: [System Programming, thesis]
tags:
  [
    System Programming,
    WhatEveryProgrammerShouldKnowAboutMemory,
    chapter6,
    Bypassing the Cache
  ]
---

# 개요

What Every Programmer Should Know About Memory에서 What Programmers Can do chapter 6을
정리하고자 함

본 챕터는,

> 성능은 컴파일러나 하드웨어가 아니라, 프로그래머의 메모리 접근 패턴에서 결정된다.

를 설명하고 있다.

본 포스터는 chapter 6.1인 캐시 우회까지 다룸.

---

# 캐시 우회(Bypassing the Cache)

데이터가 생성된 후 재사용되지 않는 경우, cache line을 먼저 읽고 캐시된 데이터를 수정하는 방식은 
성능에 악영향을 끼친다.

- 다시 필요할지 모르는 데이터를 캐시 밖으로 밀고 사용하지 않을 데이터를 캐시로 채우게 됨
- 행렬과 같이 캐시보다 큰 대규모 데이터 구조는 행렬을 채운 뒤 나중에 사용하는 경우, 마지막 요소를 채우기도 전에 

  처음 요소들이 캐시에서 밀려나서 쓰기에 대한 캐싱 효과가 없어짐

예시 : 1GB짜리 배열 초기화

```c
for (i = 0; i < N; i++)
    a[i] = 0;
```

- 배열은 한 번 쓰고 나중에나 읽음
- 그런데 CPU는 전부 캐시에 올림

결과:
- 캐시가 배열로 가득 참
- 중요한 다른 데이터 다 밀려남
- **캐시 오염(cache pollution)**

> 해당 데이터가 곧 다시 사용되지 않을 경우 캐시에 저장할 이유가 없다 !

## non-temperal write 연산

캐시 기능 컨트롤 명령어로, 데이터를 전송할 때 캐시 계층을 통하지 않고 직접 메모리에 데이터를 저장하는 기능이다.

x86, x86-64 아키텍처에서 gcc가 제공하는 intrinsic

```c
#include <emmintrin.h>
void _mm_stream_si32(int *p, int a);
void _mm_stream_si128(int *p, __m128i a);
void _mm_stream_pd(double *p, __m128d a);

#include <xmmintrin.h>
void _mm_stream_pi(__m64 *p, __m64 a);
void _mm_stream_ps(float *p, __m128 a);

#include <ammintrin.h>
void _mm_stream_sd(double *p, __m128d a);
void _mm_stream_ss(float *p, __m128 a);
```
위 명령어는 **대량의 데이터를 한 번에 처리할 때 가장 효과적**이다.

데이터는 메모리에 로드되어 하나 이상의 처리 단계를 거친 후, 다시 메모리에 기록한다.


예시

```c
#include <emmintrin.h>

void setbytes(char *p, int c)
{
    __m128i i = _mm_set_epi8(
        c, c, c, c,
        c, c, c, c,
        c, c, c, c,
        c, c, c, c);

    _mm_stream_si128((__m128i *)&p[0],  i);
    _mm_stream_si128((__m128i *)&p[16], i);
    _mm_stream_si128((__m128i *)&p[32], i);
    _mm_stream_si128((__m128i *)&p[48], i);
}
```

포인터 `p`가 정렬되어있을 때, `_mm_stream_si128`는 해당 cache line을 `c` 값으로 채운다.

write-combining 로직은 생성된 네 개의 `movntdq` 명령어를 인식하고 마지막 명령어가 실행되면,

**비로소 Write-Combining Buffer는 global visible 가능한 상태가 된다.**

위 코드의 시퀀스는
- cache line을 쓰기 전에 읽지 않고
- 곧 필요하지 않을 데이터로 캐시를 오염시키지도 않는다.

일반적으로`memset`함수가 있지만 캐시 사이즈보다 큰 데이터에 대해서는 위와 유사한 코드를 사용하는 것이 바람직하다.

하드웨어 특징
- cache line 할당 안 함
- MESI state 안 들어감
- L1/L2/L3 cache 무관
- Write-Combining Buffer에만 적재

## Write-Combining Buffer

CPU 코어 내부에 store 전용 임시 버퍼인, write-combining buffe는 store buffer 중에서도

- non-temperal store
- cache bypass store

를 위한 특수한 엔트리이다.

```
CPU Core
 ├─ Store Buffer
 │   ├─ normal store entries
 │   ├─ atomic store entries
 │   └─ write-combining entries  ←←
 │
 └─ Load Buffer
```

cache line(64B) 단위의 여러 store를 합치고 한 번에 메모리로 write하게 됨



## 2차원 배열로 이해하는 non-temperal write operation에 적합한 예시

상황
- 3000 × 3000 행렬
- 전부 0으로 초기화
- 캐시엔 안 올리고 싶음


좋은 접근(행단위, 순차)

```c
for (i = 0; i < row; i++)
  for (j = 0; j < col; j++)
    a[i][j] = 0;
```

- 메모리 순서 그대로 접근
- CPU가 예측 가능
- write-combining 잘 됨
- non-temperal write도 빠름


나쁜 접근 (열 단위)
```c
for (j = 0; j < col; j++)
  for (i = 0; i < row; i++)
    a[i][j] = 0;
```

- cache line을 벗어난 메모리 접근으로 여기저기 점프
- write-combining 불가
- RAM row 계속 전환
- non-temperal write는 더 느려짐

논문의 행렬의 초기화 테스트 결과

| 메모리 접근 방향| 행 |	열 |
| -- | -- | -- |
| 일반 쓰기 |	0.048s	| 0.127s |
| Non-temperal |	0.048s |	0.160s |

메모리 행 접근 패턴에서는 일반쓰기와 non-temperal write의 차이가 없다.

이는 프로세서가 Write-Combining을 수행하여 캐시를 경유한 일반쓰기 대비 성능 저하를 상쇄된 결과이다.


## memory barriers

> 빠르지만 무질서한 non-temperal access를 통제하는 브레이크 역할로,
> 모든 NT store가 global visibility 완료 대기한다.

명세상으로 fence는 “각 store가 자기 규칙대로 global visibility를 되도록 수행하는 명령어”다.

CPU는 빠르기 위해 다음을 마음대로 한다.

- store / load 재정렬
- store를 나중에 실제 메모리에 반영
- load를 미리 실행(speculation)

이는 CPU입장에서 문제가 없지만, 동시성 +I/O + persistence에서는 문제가 될 수 있음

예를 들어,

- non-temperal write은 아직 WCB에 있는데, 다른 코드에서 해당 데이터를 조회하려고 할 때
- `fsync` /write 시스템콜로 커널이 DRAM을 디스크에 반영할 경우 WAL프로토콜 붕괴
- 다른 CPU가 normal read할 경우 NT store global visibility 시점에서 뒤늦게 invalid되고 값이 최신화

load는 DRAM에 접근하게 되어, 최신 데이터를 보지 못하는 문제가 발생한다.

일반적으로 CPU는 아래 상황일 때 flush하게 된다.

- WCB eviction
- fence
- context switch
- interrupt
- power event

non-temperal write는 MESI프로토콜을 따르지 않고 cache coherence가 지연되기 때문에

개발자가 fence를 책임져야 한다.

따라서, non-temperal write이 끝나면 의도적으로 memory-barrier를 수행하는 것이 바람직하다.

## non-temperal read

> Non-temperal read는 “L1/L2/L3 캐시에 cache line을 넣지 않고, 
> 
> 전용 streaming load buffer에만 잠깐 담아서 레지스터로 전달하는 load”다.

일반 load의 기계적 단계는 아래와 같다.

```
1. AGU
2. L1D lookup (miss)
3. LFB(Line Fill Buffer) 할당
4. DRAM → 64B fetch
5. L3 → L2 → L1에 cache line 설치
6. Eviction 발생 가능
```
이 떄 발생하는 비용은 

- LFB 점유
- L1/L2/L3 tag update
- replacement policy 실행
- dirty line write-back 가능성
- coherence bookkeeping

이다. 쉽게 말해, "읽기 + 관리비용"


**non-temperal read** 수행 시 실제 하드웨어 경로는 아래와 같다.
```
[1] AGU (Address Generation Unit)
 ↓
[2] Line Fill Buffer (LFB) 요청
 ↓
[3] Memory Controller → DRAM
 ↓
[4] Streaming Load Buffer (SLB)
 ↓
[5] SIMD Register (XMM/YMM/ZMM)
```

non-temperal read는 cache hierachy를 완전히 우회한다.

데이터는 DRAM으로부터 코어 내에 Streaming Load Buffer로 load한다.
- 보통 2~4개의 cache line

cache 설치나 **cache eviction**, Line File Buffer 점유에 대한 압박 등이 없이 읽기만 한다.

또한, Memory-Level Parallelism이 가능해져서 같은 latency라도 bandwidth가 높다.

그래서 2 ~ 4 * 64Byte 범위에 위치한 데이터를 sequential read할 경우 일반 load 대비 높은 성능을 보인다.

추가로, cache에 streaming data가 덮히는 것을 방지함으로써, 다른 코어에 미치는 간접비용이 줄어든다.


non-temperal read 명령어는 아래와 같다.

```asm
movntdqa xmm0, [addr]     ; SSE4.1
vmovntdqa ymm0, [addr]   ; AVX
vmovntdqa zmm0, [addr]   ; AVX-512
```

컴파일러 instrinsic

```c
_mm_stream_load_si128()
```

### Streaming Load Buffer(SLB)

SLB는 “cache line 단위의 순차적인 load 요청을 감지해서,
해당 cache line들을 L1/L2 cache를 거치지 않고 임시 버퍼에 보관하며,
여러 개를 동시에 in-flight 상태로 유지해 DRAM latency를 숨기는 하드웨어 버퍼”

SLB의 Entry는 아래 정보를 포함한다.

`{ line_address, data(64B), state }`

- core 내부에 위치하고, CPU가 streaming access로 판단될 때 사용
- L1/L2 cache에 저장되지 않음
- cache replacement 정책 참여 안함
- reuse를 고려 안함
- address offset 추적 안함

핵심 역할은 
1. **Latency Hiding**
  - DRAM load latency 수백 cycle
  - SLB에 여러 line을 미리 올려둠
  - core는 stall 없이 consume
2. Cache pollution 방지
   - streaming data가 L1/L2를 밀어내지 않음

이다.

**SLB가 관리하는 하드웨어 규칙**은 아래와 같다.

허용
- line 내부 offset 접근 자유
- 같은 line 반복 접근 (SLB에 있는 동안)
- 단조 증가 line access

금지 / 비효율
- line 역행 접근
- 큰 stride jump
- reuse 기대
- long-lived register reuse

SLB entry free 규칙 
- 새로운 “더 뒤의” cache line이 들어오면, 가장 오래된 line은 더 이상 필요 없다고 판단되어 SLB에서 제거된다.
- FIFO 성향
- entry 개수 제한

**SLB depth**

동시에 in-flight로 유지할 수 있는 streaming cache line의 개수를 의미한다.

실제 동작 중 entry들은 아래와 같은 여러 state가 될 수 있고, vaild 상태로서 streaming에 기여하는 건 제한적이게 된다.

- prefetch 중
- 요청만 발행되고 아직 data 없음
- speculative load
- partial consumption

예를 들어, 물리적인 entry 개수는 4개이더라도,
```
entry0: consumed, pending free
entry1: valid line
entry2: valid line
entry3: requested, filling
```
이 상태에서 유효하게 streaming에 기여하는 건 2~3개 이며,

이게 **effective SLB depth**를 의미한다.

그래서 

- SLB entry 개수는 구조적 상한
- SLB depth는 동작 중 관측되는 유효 window 크기

로 정리된다.

### non-temperal read가 적합한 코드

SLB entry 수가 4개이고 cache line이 64B라고 할 때

(1) 가장 기본: 단순 for-loop
```c
for (size_t i = 0; i < N; i += 64) {
    __m128i v = _mm_stream_load_si128((__m128i*)(p + i));
    sum += process(v);
}
```

- 매 iteration마다 consume 직후 fill
- SLB occupancy ≈ 1
- DRAM latency 숨기기 어려움


(2) Loop unrolling + NT load 
```c
for (size_t i = 0; i < N; i += 256) {
    __m128i v0 = _mm_stream_load_si128((__m128i*)(p + i +  0));
    __m128i v1 = _mm_stream_load_si128((__m128i*)(p + i + 64));
    __m128i v2 = _mm_stream_load_si128((__m128i*)(p + i + 128));
    __m128i v3 = _mm_stream_load_si128((__m128i*)(p + i + 192));

    process(v0);
    process(v1);
    process(v2);
    process(v3);
}
```

- 연속 NT load 4개 발행
- SLB entries 채워짐
- 이후 process 동안 DRAM fetch 진행
- consume와 fill overlap


(3) Prefetch distance 조절
```c
for (size_t i = 0; i < N; i += 64) {
    _mm_prefetch(p + i + 256, _MM_HINT_NTA);

    __m128i v = _mm_stream_load_si128((__m128i*)(p + i));
    process(v);
}
```

- prefetch → SLB ahead 채움
- load → head consume
- distance = entries × line size 근처가 이상적

### non-temperal rea를 지양해야 하는 로직

**1.  random access에서 NT read**

```c
#include <smmintrin.h>

long sum_random_nt(int *A, int *idx, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        int *p = &A[idx[i]];   // random
        __m128i v = _mm_stream_load_si128((__m128i *)p);
        sum += _mm_extract_epi32(v, 0);
    }
    return sum;
}
```
- 매 iteration마다
  → cache lookup ❌
  → LFB 할당
  → DRAM row activate
  → read
- cache hit 없음
- prefetch 없음
- row buffer locality 없음

**2. reuse 가능성이 있는 데이터**

```c
void scan_and_reuse_nt(int *A, int n) {
    long sum1 = 0, sum2 = 0;

    // 1st pass
    for (int i = 0; i < n; i += 4) {
        __m128i v = _mm_stream_load_si128((__m128i *)&A[i]);
        sum1 += _mm_extract_epi32(v, 0);
    }

    // 2nd pass (reuse 기대)
    for (int i = 0; i < n; i++) {
        sum2 += A[i];  // 다시 DRAM
    }
}
```

- 1st pass에서 cache fill ❌
- 2nd pass도 전부 miss
- cache warm-up 기회를 스스로 제거한 게 됨

**3. latency-sensitive path (예: index lookup)**
```c
int btree_probe_nt(Node *node, int key) {
    __m128i v = _mm_stream_load_si128((__m128i *)node);
    if (node->key == key)
        return node->value;
    return -1;
}
```

- index probe latency = DRAM 왕복
  → query latency 증가
  → tail latency 폭증
- 단일 cache miss도 치명적인 경로
- NT read = 강제 DRAM

**4. lock / metadata 접근**
```c
struct Lock {
    volatile int state;
};

void lock_nt(struct Lock *l) {
    while (1) {
        __m128i v = _mm_stream_load_si128((__m128i *)l);
        int s = _mm_extract_epi32(v, 0);
        if (s == 0) {
            if (__sync_bool_compare_and_swap(&l->state, 0, 1))
                return;
        }
    }
}
```

- NT load는 coherence 참여가 늦고 visibility 보장 약함
- spinlock은 cache coherence 의존

➡ livelock / starvation 위험

### non-temperal read 사용 시 주의할 점

> NT load 결과를
> 
> 즉시 사용하고 오래 들고 있지 말 것
> 
> 동일 line 재접근은
> 
> SLB depth 안에서만 한다.


SFB는 “cache line 단위 FIFO”이고 소비는 “line 내부 offset 기준으로만 자유롭다”

즉, line 단위 순서는 중요하고 line 내부 접근 순서는 중요하지 않다.

또한, SLB entry는 “다음 cache line을 요청하는 순간” SLB가 가득 찬 경우 앞의 line이 free된다.

레지스터, offset, 명령 수명은 전혀 기준이 아니다.


# 요약

현대 CPU는 순차적인 non-temperal write를 매우 잘 최적화한다

최근에는 순차적인 non-temperal read도 마찬가지

한 번만 사용되는 대규모 데이터 구조 처리에 매우 유용하다

캐시는 랜덤 접근 비용의 일부만 가려줄 수 있다

이 예제에서 랜덤 접근은
RAM 구현 특성 때문에 70% 더 느렸다

RAM 구현이 바뀌기 전까지
랜덤 접근은 가능한 한 피해야 한다

---

출처

- https://people.freebsd.org/~lstewart/articles/cpumemory.pdf
