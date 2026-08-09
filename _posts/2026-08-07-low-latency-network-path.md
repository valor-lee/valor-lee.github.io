---
title: '[Low Latency Trading] Market Data가 NIC에서 Application까지 오는 길'
date: 2026-08-07 01:00:00 +09:00
categories: [computer, trading system]
published: false
tags:
  [
    low latency trading,
    market data,
    Linux networking,
    NAPI,
    RSS,
    multicast
  ]
---

# 개요

Market data packet이 server의 network port에 도착했다고 해서 application이 즉시 그 message를 읽을 수 있는 것은 아니다. Packet은 NIC의 receive queue, DMA buffer, Linux network stack과 socket receive queue를 차례로 지나간다.
```text
Wire
  → Ethernet Frame
  → NIC RX Queue와 Descriptor
  → DMA Buffer
  → MSI-X Interrupt
  → NAPI Poll
  → Ethernet / IP / UDP 또는 TCP 처리
  → Socket Receive Queue
  → recvmsg()와 User Buffer
  → Market Data Decoder
```
실제 경로는 NIC, driver, kernel version과 offload 설정에 따라 달라진다. 위 그림은 일반적인 Linux kernel network path를 이해하기 위한 지도다. 초저지연 튜닝의 출발점은 모든 기능을 끄는 것이 아니다. Packet이 어느 queue와 CPU를 거쳤고 어느 경계에서 기다리거나 유실되었는지 측정할 수 있어야 한다. 이 글의 모든 명령과 설정은 예시다. Interface 이름, driver counter, sysfs 위치와 지원 option은 system마다 다르므로 대상 kernel, NIC와 운영 환경에서 반드시 검증해야 한다.

# 학습 위치

| 항목 | 내용 |
| --- | --- |
| BFS Level | Level 1-N — Network 수신 경로 |
| 함께 읽기 | [[Low Latency Trading] Linux 실행 환경과 CPU 격리](/posts/low-latency-linux-execution/) |
| 다음 심화 | [[Device I/O] CPU는 Device와 어떻게 통신하는가](/posts/cpu-device-communication/) |
| 다음 심화 | [[Device I/O] Interrupt와 Polling, DMA는 어떻게 연결되는가](/posts/interrupt-polling-dma/) |
| 이후 선택 | Busy polling, AF_XDP와 kernel bypass |

완료 기준은 다음 경로를 자신의 말로 설명하는 것이다.

> NIC가 packet을 DMA한 뒤 application의 `recvmsg()`가 payload를 반환할 때까지 queue, CPU와 소유권이 어떻게 바뀌는가?

# 1. Protocol Layer를 먼저 구분한다

Market data는 하나의 protocol만 통과하지 않는다. 각 layer는 서로 다른 주소와 책임을 가진다.
```text
Ethernet: 같은 link의 frame과 MAC address
IP      : Host와 network 사이의 packet 전달
UDP/TCP : Process가 사용하는 port와 transport semantics
Feed    : Message type, sequence, timestamp와 recovery rule
```
Application에서 보이는 sequence number는 일반적으로 Ethernet, IP 또는 UDP가 만들어 주는 값이 아니다. Exchange 또는 feed protocol이 message ordering과 gap detection을 위해 정의한다.

# 2. Ethernet Frame

NIC는 wire에서 bit stream을 받아 Ethernet frame을 복원한다. 일반적인 frame에는 destination/source MAC address, EtherType, payload와 FCS가 포함된다.
```text
Preamble / SFD
Destination MAC
Source MAC
EtherType 또는 Length
Payload
Frame Check Sequence
```
NIC는 FCS와 frame 상태를 검사하고 오류 frame을 drop할 수 있다. Driver나 packet capture에서 FCS가 보이지 않을 수 있으며, NIC가 제거하거나 hardware에서만 검사하는 동작은 장비별로 확인해야 한다. VLAN tag, jumbo frame과 hardware timestamp도 실제 frame 처리에 영향을 줄 수 있다. MTU와 feed message 크기가 맞지 않으면 fragmentation 또는 drop을 의심해야 한다.

# 3. IP와 Transport Protocol

EtherType이 IP를 나타내면 kernel은 destination address, header와 local route를 확인해 상위 protocol로 전달한다. UDP와 TCP는 같은 IP 위에서 동작하지만 application에 제공하는 의미가 다르다.

| 항목 | UDP | TCP |
| --- | --- | --- |
| Data 경계 | Datagram 경계 유지 | Byte stream |
| 전달 보장 | 전달·중복 방지·순서 보장 없음 | 신뢰성 있는 순서화된 stream |
| Loss 반응 | Application protocol이 탐지·복구 | 재전송과 congestion control |
| 지연 특성 | Loss를 숨기지 않음 | 앞선 byte 복구가 뒤의 byte 전달을 지연 가능 |
| 대표 사용 | Market data multicast 등 | Session, recovery, order entry 등 |

실제 venue가 어떤 transport를 사용하는지는 protocol specification을 따라야 한다. “Market data는 항상 UDP이고 order는 항상 TCP다”라는 규칙은 없다.

## 3.1 UDP

UDP는 작은 protocol mechanism으로 datagram을 전달한다. RFC 768은 delivery와 duplicate protection을 보장하지 않는다고 설명한다. 따라서 receiver는 다음 상황을 고려해야 한다.

- Packet loss
- Duplicate
- Out-of-order arrival
- Datagram truncation
- Burst로 인한 socket queue overflow

한 번의 `recvmsg()`는 보통 datagram 하나의 경계를 유지한다. 제공한 buffer가 작으면 나머지 datagram이 버려질 수 있으므로 return length와 `MSG_TRUNC` 조건을 처리해야 한다.

## 3.2 TCP

TCP는 application에 ordered byte stream을 제공한다. 한 번의 `send()`가 receiver의 한 번의 `recv()`와 같은 경계로 돌아온다는 보장은 없다.
```text
Sender send(): [Header + Message A]
Sender send(): [Header + Message B]

Receiver recv(): [Header + Message A의 일부]
Receiver recv(): [나머지 A + Header + Message B]
```
Application은 length field나 delimiter를 사용해 message framing을 직접 복원해야 한다. Network loss가 발생하면 TCP는 재전송으로 byte stream의 순서를 유지한다. 이때 뒤에 도착한 data도 앞선 data의 복구를 기다릴 수 있으므로 application에는 gap 대신 latency spike로 보일 수 있다.

# 4. Multicast는 복제 방식이다

IP multicast에서는 sender가 group address로 datagram을 보내고 network가 가입 receiver에게 복제한다.
```text
Publisher
    ↓ one multicast stream
Network / Switch
    ├─ Receiver A
    ├─ Receiver B
    └─ Receiver C
```
Receiver는 group과 interface를 지정해 membership을 요청한다. IPv4 socket에서는 `IP_ADD_MEMBERSHIP` 같은 option을 사용할 수 있다.
```text
Socket 생성
  → Local address와 port bind
  → Multicast group 가입
  → Datagram receive
```
Group 가입은 application API만의 문제가 아니다. Host의 IGMP 동작, switch의 IGMP snooping, multicast route와 VLAN 구성을 함께 확인해야 한다. Feed가 source-specific multicast를 사용하는지, A/B redundant line을 제공하는지와 recovery channel을 어떻게 정의하는지도 venue specification에 달려 있다.

# 5. RX Descriptor와 DMA Buffer

Driver는 NIC가 packet을 쓸 수 있도록 RX buffer를 준비하고 descriptor ring에 device가 사용할 address와 상태를 기록한다.
```text
Driver
  1. RX buffer 준비
  2. DMA mapping
  3. Descriptor에 address 제공
  4. Descriptor ownership을 NIC에 전달

NIC
  5. Packet 수신
  6. Payload를 host memory에 DMA write
  7. Length와 status 기록
  8. Descriptor ownership을 host에 반환
```
Descriptor는 packet payload 자체가 아니라 buffer 위치, 길이와 상태를 표현하는 metadata다. NIC가 사용할 준비가 된 descriptor가 부족하면 새 packet을 저장할 곳이 없어 drop이 발생할 수 있다. Ring 크기를 크게 하면 burst 흡수에는 도움이 될 수 있지만 queueing delay와 memory 사용량도 증가한다. DMA가 CPU를 거치지 않고 wire의 data를 application object로 바로 완성한다는 뜻은 아니다. DMA 이후에도 driver와 protocol stack이 packet을 분류하고 socket으로 전달해야 한다.

# 6. Multi-Queue와 RSS

현대 NIC는 여러 RX queue를 제공할 수 있다. RSS(Receive Side Scaling)는 NIC가 header의 hash 등을 사용해 flow를 RX queue에 분산하는 hardware mechanism이다.
```text
Flow Hash
  ├─ RX Queue 0 → MSI-X 0 → CPU 0
  ├─ RX Queue 1 → MSI-X 1 → CPU 1
  └─ RX Queue 2 → MSI-X 2 → CPU 2
```
같은 flow를 같은 queue로 보내면 processing parallelism을 얻으면서 flow 내부 순서가 바뀔 가능성을 줄일 수 있다. 그러나 하나의 UDP multicast flow는 hash 관점에서 하나의 queue에 계속 들어갈 수 있다. Queue 수를 늘렸다고 하나의 feed가 자동으로 여러 core에 분산되지는 않는다. NIC가 사용하는 hash field, indirection table과 flow steering rule은 다음 종류의 명령으로 관찰할 수 있다.
```bash
ethtool -l DEVICE
ethtool -x DEVICE
ethtool -n DEVICE
```
Option 지원과 출력 형식은 driver마다 다르다. 먼저 조회한 뒤 변경 가능 여부를 확인한다.

# 7. MSI-X와 Interrupt Moderation

NIC는 descriptor completion을 만든 뒤 MSI-X interrupt로 host에 event를 알릴 수 있다. Multi-queue NIC는 queue별 vector를 제공해 여러 CPU로 interrupt를 분산할 수 있다.
```text
Packet 1 완료 ─┐
Packet 2 완료 ─┼─ Interrupt 한 번
Packet 3 완료 ─┘
```
Interrupt moderation 또는 coalescing은 여러 event를 모아 interrupt 수를 줄인다. Throughput과 CPU 효율에는 유리할 수 있지만 첫 packet이 기다리는 시간이 늘어날 수 있다. 현재 설정과 통계의 예시는 다음처럼 확인한다.
```bash
ethtool -c DEVICE
cat /proc/interrupts
cat /proc/irq/IRQ_NUMBER/effective_affinity_list
```
Adaptive moderation은 traffic에 따라 값을 바꿀 수 있다. Benchmark 중 설정이 고정되어 있다고 가정하지 말고 실제 counter와 configuration을 함께 기록한다.

# 8. NAPI는 Interrupt와 Polling을 연결한다

Linux network driver는 일반적으로 packet마다 전체 protocol stack을 hard IRQ context에서 실행하지 않는다. 기본적인 NAPI 흐름은 다음과 같다.
```text
NIC Interrupt
  → Driver가 NAPI instance schedule
  → 해당 queue의 interrupt를 mask 또는 억제
  → NAPI poll이 budget만큼 RX completion 처리
  → 남은 일이 있으면 다시 poll
  → 모두 처리하면 interrupt를 다시 enable
```
NAPI poll은 보통 software interrupt context에서 실행되며 driver가 queue의 packet을 batch로 처리하게 한다. `budget`은 한 번의 poll에서 처리할 RX packet 수를 제한해 특정 queue가 CPU를 독점하지 않도록 돕는다. 계속 budget을 소진하면 추가 poll이 필요하고, 높은 부하에서는 softirq 처리 지연이나 `ksoftirqd` 실행이 관찰될 수 있다. NAPI는 “interrupt 방식” 또는 “polling 방식” 중 하나를 고르는 단순한 switch가 아니다. Interrupt가 초기 event를 알리고 NAPI가 completion을 poll하는 hybrid 구조가 일반적이다.

# 9. skb와 Protocol Stack

Driver는 RX buffer의 data를 Linux network stack이 다룰 수 있는 packet representation에 연결한다. 흔히 `sk_buff`, 즉 skb가 metadata와 packet data를 표현한다. 개념적인 receive 처리는 다음과 같다.
```text
Driver / NAPI
  → Ethernet header 처리
  → VLAN과 packet type 확인
  → IP header와 local delivery 판단
  → UDP socket demultiplex 또는 TCP state 처리
  → 대상 socket receive queue에 enqueue
```
실제 driver는 page pool, GRO와 checksum offload 등을 사용할 수 있어 buffer 구조와 함수 호출은 다를 수 있다. GRO(Generic Receive Offload)는 여러 packet을 더 큰 단위로 묶어 protocol processing 비용을 줄일 수 있다. Batching은 throughput에 유리하지만 message가 application에 보이는 시점과 packet 형태에 영향을 줄 수 있으므로 무조건 켜거나 끄지 않고 측정해야 한다.

# 10. Socket Receive Queue와 Application Copy

Protocol stack이 destination socket을 찾으면 packet 또는 data를 socket receive queue에 넣고 대기 중인 thread를 깨울 수 있다.
```text
Kernel Socket Receive Queue
  → Readiness notification
  → Application의 recvfrom() / recvmsg()
  → User-space buffer
  → Feed decoder와 sequence check
```
일반적인 UDP/TCP socket receive에서는 `recvmsg()` 계열 호출이 kernel이 관리하는 packet data를 application buffer로 복사한다. Driver와 kernel 기능에 따라 세부 copy 경로와 zero-copy 지원은 다를 수 있다. `epoll`은 file descriptor의 readiness를 기다리는 interface다. `epoll`을 사용한다고 NIC-to-application data path가 자동으로 zero-copy가 되지는 않는다. Socket receive buffer가 가득 찬 상태에서 application이 충분히 빨리 읽지 못하면 packet이 drop될 수 있다. Buffer를 크게 하면 burst 내성은 늘 수 있지만 backlog가 길어져 오래된 market data를 늦게 처리하는 문제를 만들 수 있다.

# 11. Sequence Number가 Loss를 드러낸다

UDP 자체는 feed message의 sequence를 관리하지 않는다. Market data protocol이 sequence number를 제공한다면 application은 이를 이용해 gap, duplicate와 out-of-order를 구분한다.
```text
Expected: 1001
Received: 1001 → 정상, next = 1002
Received: 1004 → 1002~1003 gap 가능
Received: 1001 → duplicate 또는 late packet 가능
```
Sequence gap은 문제가 존재한다는 강한 증거지만 어느 layer에서 drop되었는지 자동으로 알려주지는 않는다. 가능한 지점은 다음과 같다.

| 지점 | 가능한 원인 | 확인할 증거 |
| --- | --- | --- |
| Wire 이전 | Publisher 또는 upstream loss | Feed A/B 비교, venue status |
| Link/NIC | CRC, missed packet, no buffer | NIC와 switch counter |
| RX ring | Descriptor 부족 | Driver-specific RX drop counter |
| Network stack | Backlog와 protocol drop | softnet, IP/UDP counter |
| Socket queue | Application receive 지연 | socket memory와 overflow counter |
| Application | Decoder 또는 internal queue drop | Feed sequence와 app counter |

Linux의 `SO_RXQ_OVFL` option은 socket이 생성된 뒤 drop한 packet 수를 ancillary message로 받을 수 있게 한다. 이것도 전체 network path의 모든 loss를 세는 counter는 아니다. Recovery는 protocol에 따라 retransmission request, snapshot 재동기화, A/B feed 선택 또는 session reset을 사용할 수 있다. Gap을 발견한 뒤 stale state로 계속 주문할지 중단할지는 trading risk policy의 일부다.

# 12. RSS, RPS와 RFS

이름이 비슷하지만 packet을 움직이는 위치가 다르다.

| 기능 | 실행 위치 | 선택 대상 |
| --- | --- | --- |
| RSS | NIC hardware | RX queue와 초기 처리 CPU |
| RPS | Kernel software | 상위 protocol processing CPU |
| RFS | Kernel software + flow 정보 | 소비 application에 가까운 CPU |

RPS(Receive Packet Steering)는 RSS의 software 형태로 볼 수 있다. Driver가 packet을 stack에 올린 뒤 다른 CPU의 backlog queue로 보내 protocol processing을 분산할 수 있다. 다른 CPU로 보낼 때 inter-processor interrupt와 cache 이동 비용이 생길 수 있다. 이미 NIC RSS가 적절한 queue와 CPU를 선택한다면 RPS가 오히려 latency를 늘릴 수 있다. RFS(Receive Flow Steering)는 application이 flow를 소비하는 CPU 정보를 이용해 processing CPU를 맞추려 한다. Accelerated RFS는 지원 NIC와 driver에서 hardware steering까지 연결할 수 있다. 초저지연 구성에서는 다음 묶음을 하나의 locality chain으로 본다.
```text
NIC Port / NUMA Node
  → RX Queue
  → MSI-X IRQ CPU
  → NAPI CPU
  → Socket을 읽는 Thread CPU
  → Receive Buffer의 Memory Node
```
RSS, RPS와 RFS를 동시에 켜기 전에 packet이 실제로 어느 CPU를 거치는지 확인한다.

# 13. Drop Counter를 계층별로 본다

하나의 `dropped` 숫자만으로 원인을 결론 내리면 안 된다.
```bash
ip -s link show dev DEVICE
ethtool -S DEVICE
nstat -az
ss -u -a -m
cat /proc/net/softnet_stat
cat /proc/softirqs
```
주의할 점은 다음과 같다.

- `ethtool -S` counter 이름과 의미는 driver마다 다르다.
- `/proc/net/softnet_stat` field 해석은 kernel version을 확인해야 한다.
- Socket queue 상태는 관찰 순간의 snapshot이다.
- NIC hardware drop과 UDP socket drop은 서로 다른 사건이다.
- Counter가 wrap 또는 reset되는 조건을 확인해야 한다.

Application counter에는 최소한 received datagram, decoded message, sequence gap, duplicate, out-of-order, recovery와 internal queue drop을 따로 기록하는 것이 좋다.

# 14. Latency Timestamp를 경계별로 둔다

End-to-end latency 하나만 측정하면 어느 구간이 느린지 알기 어렵다.
```text
T0: Exchange 또는 feed packet timestamp
T1: NIC hardware RX timestamp
T2: Kernel software receive timestamp
T3: recvmsg() 반환 직후
T4: Decode와 book update 완료
T5: Strategy decision 완료
```
각 차이는 다른 질문에 답한다.

| 구간 | 의미 |
| --- | --- |
| T1 - T0 | 외부 network와 clock alignment 포함 |
| T2 - T1 | NIC, driver와 초기 kernel path |
| T3 - T2 | Socket queue 대기와 wake-up/copy |
| T4 - T3 | Decoder와 order book update |
| T5 - T4 | Strategy 계산 |

`SO_TIMESTAMPING`은 software와 hardware를 포함한 여러 timestamp source를 지원한다. Hardware timestamp가 있다고 곧바로 exchange-to-host one-way latency를 정확히 계산할 수 있는 것은 아니다. Clock domain, PTP synchronization, NIC PHC와 timestamp가 찍히는 정확한 지점을 확인해야 한다. 서로 다른 clock을 보정 없이 빼면 정밀한 숫자처럼 보이는 잘못된 결과가 나온다.

# 15. 측정 순서

## 15.1 Packet 자체를 검증한다

Source/destination, VLAN, multicast group, port, message size와 sequence를 확인한다. `tcpdump` 같은 capture tool도 CPU와 buffer를 사용하고 packet을 놓칠 수 있으므로 capture 결과만 절대 기준으로 사용하지 않는다.

## 15.2 Queue와 CPU Mapping을 그린다

RX queue, RSS table, MSI-X vector, IRQ CPU, NAPI와 application CPU를 하나의 표로 기록한다.

## 15.3 모든 Drop 지점의 Baseline을 수집한다

NIC, network stack, socket과 application counter를 측정 시작과 끝에 함께 읽는다.

## 15.4 Burst를 재현한다

평균 packet rate만 맞추지 말고 market open이나 news event처럼 짧은 burst를 재현한다. Ring과 socket buffer는 평균보다 burst에서 먼저 포화될 수 있다.

## 15.5 한 변수만 바꾼다

IRQ affinity, queue 수, interrupt moderation, socket buffer, thread affinity와 batch size를 한 번에 하나씩 변경한다.

## 15.6 Tail과 Loss를 함께 본다

Latency가 낮아져도 sequence gap이 늘었다면 성공한 튜닝이 아니다. p50, p99, p99.9, maximum, packet loss와 CPU 사용량을 함께 기록한다.

# 16. Busy Polling과 Kernel Bypass 예고

NAPI busy polling은 application이 blocking receive 또는 polling API 안에서 NIC event를 확인해 interrupt를 기다리는 시간을 줄이는 방법이다. `SO_BUSY_POLL`과 관련 sysctl이 있지만 CPU cycle과 전력 사용량이 증가한다. NIC와 driver가 지원해야 하며 socket이 마지막으로 packet을 받은 NAPI instance와 CPU 배치도 중요하다.
```text
Interrupt-driven
  NIC event → IRQ → NAPI → Socket wake-up

Busy polling
  Application → NAPI poll 시도 → Packet 발견
```
Kernel bypass는 AF_XDP 또는 DPDK 같은 방식으로 kernel의 일반 socket path 일부를 우회하고 queue와 buffer를 user space에 더 직접 노출한다. 단계가 줄어드는 대신 application이 다음 책임을 더 많이 가진다.

- RX/TX ring ownership
- Huge page 또는 pinned memory
- Polling core 운영
- Packet parsing과 protocol 처리
- Drop, backpressure와 recovery
- Observability와 장애 격리

일반 socket path에서 bottleneck 위치를 측정하지 않은 채 kernel bypass로 이동하면 문제 원인과 운영 비용을 함께 숨길 수 있다. Busy polling과 kernel bypass는 후속 글에서 별도로 다룬다.

# 17. 자주 생기는 오해

## 17.1 NIC가 DMA하면 Zero-Copy다

NIC가 host memory에 payload를 DMA하는 것과 application이 copy 없이 그 buffer를 읽는 것은 다른 문제다. 일반 socket path에는 protocol processing과 user buffer copy가 있을 수 있다.

## 17.2 RSS Queue를 늘리면 하나의 Multicast Feed가 여러 Core로 분산된다

RSS는 flow hash를 기준으로 queue를 선택한다. 하나의 flow는 같은 queue에 유지될 수 있으므로 실제 indirection과 packet distribution을 확인해야 한다.

## 17.3 UDP Sequence Gap은 NIC Drop을 뜻한다

Publisher부터 application internal queue까지 어느 지점에서도 gap이 발생할 수 있다. 계층별 counter로 범위를 좁혀야 한다.

## 17.4 Socket Buffer는 클수록 좋다

큰 buffer는 burst 흡수에 도움을 주지만 stale data의 queueing time을 늘릴 수 있다. Loss와 age를 함께 측정해야 한다.

## 17.5 Interrupt Coalescing을 끄면 항상 가장 빠르다

Interrupt 수가 지나치게 늘면 CPU가 packet processing보다 interrupt handling에 더 많은 시간을 쓸 수 있다. Traffic pattern별 측정이 필요하다.

# 18. 완료 기준

다음 질문에 자신의 문장과 실제 counter로 답할 수 있어야 한다.

1. Ethernet, IP, UDP/TCP와 feed protocol은 각각 어떤 경계를 제공하는가?
2. RX descriptor와 RX payload buffer는 무엇이 다른가?
3. NIC가 DMA한 뒤 CPU는 completion을 어떻게 발견하는가?
4. MSI-X interrupt와 NAPI poll은 어떻게 연결되는가?
5. Packet은 어느 시점에 socket receive queue에 들어가는가?
6. 일반 socket receive에서 application buffer까지 어떤 copy가 남을 수 있는가?
7. UDP sequence gap이 NIC drop과 같은 뜻이 아닌 이유는 무엇인가?
8. RSS, RPS와 RFS가 packet 처리 CPU를 고르는 위치는 어떻게 다른가?
9. RX queue, IRQ, NAPI, application과 NUMA node를 하나의 그림으로 그릴 수 있는가?
10. Hardware timestamp와 application timestamp로 network와 software latency를 분리할 수 있는가?

# 정리

Market data receive latency는 socket API 하나의 비용이 아니다.
```text
Protocol Semantics
  + NIC Queue와 DMA
  + MSI-X와 NAPI
  + Linux Protocol Stack
  + Socket Queue와 User Copy
  + Application Sequence와 Decode
```
UDP는 loss를 복구해 주지 않고 TCP는 loss를 latency로 숨길 수 있다. Multicast는 여러 receiver에게 효율적으로 복제하지만 application은 membership, sequence와 recovery policy를 이해해야 한다. RSS, RPS, IRQ affinity와 buffer 크기를 바꾸기 전에 packet 경로와 drop counter를 계층별로 그린다. 최적화의 기준은 평균 packet rate가 아니라 tail latency, loss, CPU 비용과 stale data의 age다.

# 참고 자료

- [Linux Kernel — NAPI](https://docs.kernel.org/networking/napi.html)
- [Linux Kernel — Scaling in the Linux Networking Stack](https://docs.kernel.org/networking/scaling.html)
- [Linux Kernel — Dynamic Interrupt Moderation](https://docs.kernel.org/networking/net_dim.html)
- [Linux Kernel — Timestamping](https://docs.kernel.org/networking/timestamping.html)
- [Linux Kernel — Dynamic DMA Mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html)
- [Linux Kernel — SMP IRQ affinity](https://docs.kernel.org/core-api/irq/irq-affinity.html)
- [Linux man-pages — socket(7)](https://man7.org/linux/man-pages/man7/socket.7.html)
- [Linux man-pages — recv(2)](https://man7.org/linux/man-pages/man2/recv.2.html)
- [Linux man-pages — udp(7)](https://man7.org/linux/man-pages/man7/udp.7.html)
- [Linux man-pages — tcp(7)](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [RFC 768 — User Datagram Protocol](https://www.rfc-editor.org/rfc/rfc768.html)
- [RFC 9293 — Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc9293.html)
- [RFC 1112 — Host Extensions for IP Multicasting](https://www.rfc-editor.org/rfc/rfc1112.html)
- [RFC 3376 — Internet Group Management Protocol, Version 3](https://www.rfc-editor.org/rfc/rfc3376.html)
