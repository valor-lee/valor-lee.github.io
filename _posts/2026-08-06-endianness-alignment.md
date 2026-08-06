---
title: '[Computer Architecture] Endianness와 Alignment'
date: 2026-08-06 00:30:00 +09:00
categories: [computer, architecture]
published: false
tags:
  [
    endianness,
    alignment,
    padding,
    data layout,
    ABI
  ]
---

# 개요

Memory는 address가 붙은 byte의 배열로 볼 수 있다.

하지만 program은 2 byte, 4 byte, 8 byte처럼 여러 byte로 구성된 값을 사용한다.

```c
uint32_t value = 0x12345678;
```

이 값을 memory의 네 byte에 어떤 순서로 저장할지를 endianness가 결정한다.

값을 어느 address 경계에 배치할지를 alignment가 결정한다.

두 개념은 binary file, network protocol, MMIO register, DMA descriptor와 C structure layout을 이해하는 데 필요하다.

# 1. Byte와 Multi-Byte Value

Memory의 각 address는 일반적으로 한 byte를 가리킨다.

32-bit 정수는 연속된 네 byte를 사용한다.

```text
Address A     : 1 byte
Address A + 1 : 1 byte
Address A + 2 : 1 byte
Address A + 3 : 1 byte
```

문제는 정수의 가장 중요한 byte와 가장 덜 중요한 byte를 어느 address에 놓을지다.

# 2. Big-Endian과 Little-Endian

`0x12345678`을 address `0x1000`부터 저장한다고 생각해보자.

## 2.1 Big-Endian

가장 중요한 byte인 `0x12`를 가장 낮은 address에 저장한다.

```text
Address   Value
0x1000    0x12
0x1001    0x34
0x1002    0x56
0x1003    0x78
```

## 2.2 Little-Endian

가장 덜 중요한 byte인 `0x78`을 가장 낮은 address에 저장한다.

```text
Address   Value
0x1000    0x78
0x1001    0x56
0x1002    0x34
0x1003    0x12
```

Endianness는 byte의 bit 순서를 뒤집는 개념이 아니다.

Multi-byte value를 구성하는 byte들의 memory 배치 순서를 의미한다.

# 3. Register의 숫자와 Memory Layout

CPU register에서 `0x12345678`이라는 숫자의 의미가 endianness에 따라 달라지는 것은 아니다.

차이는 이 값을 memory에 저장하거나 memory에서 다시 조합할 때 나타난다.

```text
Register value: 0x12345678
       ↓ store
Memory bytes: architecture의 byte order에 따라 배치
```

# 4. Network Byte Order

서로 다른 endianness를 사용하는 system이 binary integer를 그대로 주고받으면 값을 다르게 해석할 수 있다.

Network protocol은 multi-byte integer의 byte order를 명시해야 한다.

전통적인 network byte order는 big-endian이다.

```c
uint32_t network_value = htonl(host_value);
uint32_t host_value = ntohl(network_value);
```

Protocol의 field를 처리할 때 host architecture의 endianness를 가정하지 않고 명시적인 encode/decode를 수행해야 한다.

# 5. Binary File과 Storage Format

Binary file format도 integer field의 byte order를 정의해야 한다.

다음 구조체를 그대로 file에 쓰는 방식은 이식성이 낮다.

```c
struct header {
    uint16_t version;
    uint32_t length;
};
```

문제는 endianness만이 아니다.

- Compiler가 삽입한 padding
- Type size
- Alignment
- ABI
- Structure layout

Portable format은 field별 byte order와 width를 명시하고 직접 serialize해야 한다.

# 6. MMIO와 Endianness

Device register가 정의하는 byte order와 CPU의 native byte order가 다를 수 있다.

Driver는 specification과 OS accessor가 제공하는 변환 규칙을 따라야 한다.

```text
CPU Native Value
    ↓ 필요하면 byte-order 변환
Bus / Device Register Representation
```

Register를 단순 C pointer로 읽고 host integer처럼 해석하면 architecture에 따라 잘못된 결과가 생길 수 있다.

# 7. Alignment

Alignment는 object의 시작 address가 만족해야 하는 경계 조건이다.

예를 들어 4-byte alignment는 시작 address가 4의 배수여야 한다는 뜻이다.

```text
Aligned address:   0x1000, 0x1004, 0x1008
Unaligned address: 0x1001, 0x1002, 0x1003
```

C에서는 `_Alignof` 또는 `alignof`로 type의 alignment requirement를 확인할 수 있다.

```c
#include <stdalign.h>
#include <stdio.h>

printf("%zu\n", alignof(uint64_t));
```

# 8. Alignment가 필요한 이유

Hardware는 정렬된 access를 더 단순하고 효율적으로 처리할 수 있다.

하나의 access가 자연스러운 boundary를 넘으면 여러 memory transaction이 필요할 수 있다.

```text
Aligned 8-byte access
+-----------------------+
|       8 bytes         |
+-----------------------+

Unaligned 8-byte access
       +-----------------------+
Line A | part 1                |
       +-----------+-----------+
Line B             | part 2    |
                   +-----------+
```

특히 cache line이나 page boundary를 넘으면 추가 cache line 접근 또는 page translation이 필요할 수 있다.

# 9. Unaligned Access

Unaligned access의 동작은 architecture와 instruction에 따라 다르다.

- Hardware가 처리하지만 더 느릴 수 있음
- 여러 access로 분할될 수 있음
- 특정 instruction에서 fault가 발생할 수 있음
- Device register에서는 허용되지 않을 수 있음
- Atomicity 보장이 깨질 수 있음

“x86에서는 unaligned access가 가능하다”는 사실을 모든 architecture와 모든 memory type에 일반화하면 안 된다.

# 10. Structure Padding

Compiler는 각 field의 alignment와 structure 자체의 alignment를 맞추기 위해 field 사이와 끝에 padding을 넣을 수 있다.

```c
struct example {
    uint8_t  type;
    uint32_t length;
    uint16_t flags;
};
```

개념적인 layout은 다음과 같을 수 있다.

```text
+------+---------+--------+-------+--------------+
| type | padding | length | flags | tail padding |
+------+---------+--------+-------+--------------+
```

정확한 layout은 ABI와 compiler 규칙에 따라 결정된다.

`sizeof(struct example)`이 field 크기의 단순 합과 같다고 가정하면 안 된다.

# 11. Field 순서와 Memory 사용량

Field 순서를 바꾸면 padding을 줄일 수 있다.

```c
struct compact_example {
    uint32_t length;
    uint16_t flags;
    uint8_t  type;
};
```

하지만 다음 사항도 함께 고려해야 한다.

- ABI 또는 file format 호환성
- Cache locality
- 자주 함께 접근하는 field
- False sharing
- 외부 protocol이 요구하는 layout

단순히 `sizeof`를 줄이는 것이 항상 최적은 아니다.

# 12. Packed Structure

Compiler extension이나 attribute로 padding을 제거할 수 있다.

그러나 packed structure의 field는 unaligned address에 놓일 수 있다.

```text
Padding 감소
  ↔ Unaligned access와 portability 문제 증가
```

Hardware descriptor나 protocol header를 표현할 때 packed structure를 사용할 수 있지만 specification, compiler behavior와 access 방법을 확인해야 한다.

Portable code에서는 byte buffer와 명시적인 encode/decode가 더 안전할 수 있다.

# 13. Alignment와 Atomic Operation

Atomic type은 구현이 요구하는 alignment를 만족해야 한다.

Misaligned atomic object는 언어 규칙을 위반하거나 hardware가 하나의 atomic transaction으로 처리할 수 없을 수 있다.

```text
Aligned atomic access
  → 하나의 자연스러운 hardware 단위로 처리 가능

Misaligned atomic access
  → 여러 boundary를 넘을 수 있음
  → atomicity 또는 지원 여부 문제
```

`memcpy`나 packed buffer 안의 byte를 atomic pointer로 강제 변환하면 alignment와 object lifetime 문제를 함께 만들 수 있다.

# 14. Alignment와 Cache Line

Alignment는 correctness뿐 아니라 성능에도 사용된다.

## 14.1 False Sharing 방지

서로 다른 thread가 자주 변경하는 값을 별도 cache line에 배치할 수 있다.

```c
struct counter {
    alignas(64) atomic_uint_fast64_t value;
};
```

실제 cache line size와 object 배열의 layout을 확인해야 한다.

## 14.2 SIMD와 DMA

일부 vector instruction, DMA engine 또는 device descriptor는 특정 alignment를 요구하거나 정렬된 buffer에서 더 효율적으로 동작할 수 있다.

요구 조건은 architecture와 device specification을 따라야 한다.

# 15. Alignment와 Page Boundary

Buffer가 page boundary를 넘으면 하나의 논리적 buffer가 여러 physical page에 mapping될 수 있다.

Device가 연속적인 physical memory만 지원한다면 문제가 될 수 있다.

Scatter-gather와 IOMMU는 여러 physical page를 device가 처리할 수 있는 형태로 표현하거나 mapping하는 데 사용될 수 있다.

# 16. 안전한 Binary Decode

외부에서 받은 byte buffer를 structure pointer로 강제 변환하는 방식은 다음 문제를 가질 수 있다.

```c
const struct header *header = (const struct header *)buffer;
```

- Buffer alignment가 structure 요구사항을 만족하지 않을 수 있음
- Endianness가 다를 수 있음
- Buffer 길이가 부족할 수 있음
- Structure padding이 protocol layout과 다를 수 있음
- Strict aliasing과 object representation 문제

안전한 접근은 길이를 먼저 검사하고 byte를 명시적으로 decode하거나 적절한 임시 object에 `memcpy`한 뒤 byte order를 변환하는 것이다.

# 17. 자주 생기는 오해

## 17.1 Little-Endian은 bit 순서를 뒤집는다

Endianness는 multi-byte value를 구성하는 byte의 address 순서다.

## 17.2 Structure 크기는 field 크기의 합이다

Alignment를 위한 padding이 포함될 수 있다.

## 17.3 Packed를 사용하면 항상 효율적이다

Memory 크기는 줄어도 unaligned access와 portability 비용이 생길 수 있다.

## 17.4 Unaligned Access가 가능한 CPU에서는 신경 쓰지 않아도 된다

성능, atomicity, SIMD, MMIO와 다른 architecture 호환성을 고려해야 한다.

## 17.5 Endianness는 Network에서만 필요하다

Binary file, device register, DMA descriptor, storage metadata와 cross-platform data exchange에도 필요하다.

# 정리

Endianness는 multi-byte value의 byte 배치 순서를 정의한다.

Alignment는 object를 배치하고 접근할 address 경계를 정의한다.

```text
Endianness
  → Byte를 어떤 순서로 해석하는가

Alignment
  → Value를 어느 address 경계에 배치하는가
```

두 개념은 C structure, protocol, binary file과 device interface를 연결할 때 함께 확인해야 한다.

다음 글에서는 memory access 시간이 CPU와 memory의 물리적 위치에 따라 달라지는 NUMA 구조를 살펴본다.
