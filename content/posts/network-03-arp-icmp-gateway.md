---
title: "네트워크 기초 ③ ARP·ICMP·Gateway — IP/MAC 매핑과 라우팅 동작"
date: 2026-06-16
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "ARP", "ICMP", "Gateway", "라우팅"]
summary: "IP↔MAC을 잇는 ARP, 진단 프로토콜 ICMP, 외부로 나가는 Gateway 동작을 정리한다. 서버/네트워크 지식 시리즈 3편."
---

> **개요** — L3 통신은 IP 주소로 목적지를 정하지만, 실제로 프레임을 전달할 때는 **L2 MAC 주소**가 필요하다. 이 글은 IP↔MAC을 잇는 ARP, 진단 프로토콜 ICMP, 그리고 다른 네트워크로 나가기 위한 Gateway의 동작 원리를 정리한다.

---

## 1. ARP (Address Resolution Protocol)

> 같은 LAN에서 통신할 때 **IP 주소에 대응되는 MAC 주소를 찾는 프로토콜**.

### 1.1 왜 필요한가

- L3 통신은 IP로 결정되지만, 실제 프레임 전달은 L2 MAC으로 이뤄짐
- 같은 LAN에서는 **목적지 IP → 목적지 MAC** 매핑이 필요
- 이 매핑을 찾는 도구가 ARP

### 1.2 동작 방식

```
[Host A] 192.168.0.10 — 192.168.0.20에게 보내고 싶음, MAC 모름

1. ARP Request (Broadcast)
   "192.168.0.20의 MAC 주소가 뭐야?"
   Src MAC: AA:AA:AA:AA:AA:AA
   Dst MAC: FF:FF:FF:FF:FF:FF  ← Broadcast

2. ARP Reply (Unicast)
   "내 MAC은 BB:BB:BB:BB:BB:BB야"
   Src MAC: BB:BB:BB:BB:BB:BB
   Dst MAC: AA:AA:AA:AA:AA:AA
```

> **핵심**
> - **ARP Request**: Broadcast (LAN 내 모든 호스트에 전송)
> - **ARP Reply**: Unicast (요청한 호스트에게만 응답)
> - 이 때문에 **ARP는 LAN 내부에서만 동작**한다 (Broadcast Domain 한정)

### 1.3 ARP Table

- 한 번 학습한 IP-MAC 매핑은 캐시(ARP Table)에 저장되어 재사용
- 일정 시간 후 만료(보통 수십 초 ~ 수 분)

```bash
# Linux
ip neigh show
arp -an

# Windows
arp -a
```

---

## 2. ICMP (Internet Control Message Protocol)

> 네트워크 진단·오류 통보용 L3 프로토콜.
> IP 페이로드 안에 캡슐화되어 전송된다 (IP Protocol Number 1).

### 2.1 주요 메시지

| 타입 | 이름 | 용도 |
| --- | --- | --- |
| 0 | Echo Reply | Ping 응답 |
| 3 | Destination Unreachable | 목적지 도달 불가 (라우팅 실패, 포트 닫힘 등) |
| 5 | Redirect | 더 좋은 경로 안내 |
| 8 | Echo Request | Ping 요청 |
| 11 | Time Exceeded | TTL 만료 (traceroute 동작 원리) |

### 2.2 Ping 동작

```
[Host A]  ──── ICMP Echo Request (Type 8) ────▶  [Host B]
[Host A]  ◀──── ICMP Echo Reply   (Type 0) ────  [Host B]
```

- 응답 도착 시간(RTT)으로 네트워크 지연 측정
- 응답 없음 = 통신 불가 (단, 보안 정책으로 ICMP 차단되는 경우도 있음 → ICMP 실패 ≠ 항상 장애)

### 2.3 Traceroute 동작 (참고)

- TTL을 1부터 증가시키며 패킷 전송
- 각 라우터가 TTL=0으로 만든 후 **ICMP Time Exceeded** 응답
- 이를 통해 경로상 라우터들을 차례로 식별

---

## 3. Gateway

> 자기 자신이 속한 네트워크 **외부**의 대상과 통신하기 위해 거쳐야 하는 L3 장비.
> 보통 라우터, L3 스위치가 Gateway 역할을 수행한다.

### 3.1 동일 네트워크 vs 다른 네트워크 판단

송신 호스트는 목적지 IP와 자신의 IP를 **Subnet Mask**로 비교한다.

```
자신의 IP    : 192.168.0.10/24
Subnet Mask : 255.255.255.0  (앞 24bit가 Network)

[Case 1] 목적지: 192.168.0.20
  → 192.168.0.0/24 동일 네트워크 → LAN 통신 (Switch, MAC 기반)

[Case 2] 목적지: 10.0.0.5
  → 192.168.0.0/24와 다른 네트워크 → WAN 통신 (Gateway 경유, Router 기반)
```

### 3.2 동일 네트워크 통신 흐름

```
1. 목적지 IP의 MAC을 ARP Table에서 검색
   - 없으면 ARP Request로 학습
2. L2 헤더의 Dst MAC = 목적지 호스트의 MAC
3. Switch가 MAC 주소 기반으로 프레임 전달
```

### 3.3 외부 네트워크 통신 흐름 (Gateway 경유)

```
1. Gateway IP의 MAC을 ARP Table에서 검색
   - 없으면 ARP Request로 학습 (※ 목적지 IP가 아니라 Gateway IP의 MAC!)
2. L2 헤더의 Dst MAC = Gateway의 MAC
   L3 헤더의 Dst IP  = 최종 목적지 IP (변하지 않음)
3. Gateway(라우터)가 받으면 L2 디캡슐화 → L3 확인 → 라우팅 결정 → 다시 L2 캡슐화
```

> **핵심 규칙**
> - **L3 헤더의 Src/Dest IP는 종단 간 유지** (NAT 제외)
> - **L2 헤더의 Src/Dest MAC은 매 홉마다 Rewrite**

### 3.4 라우터의 MAC Rewrite 동작

```
[Host A 192.168.0.10] ─ MAC: AA
       │ L2(Dst=R1_in) | L3(Dst=8.8.8.8) | DATA
       ▼
[Router R1]
   - L2 디캡슐화 → L3 헤더 확인
   - 목적지 IP 8.8.8.8 → 라우팅 테이블 조회 → 다음 홉 결정
   - 내보내는 인터페이스의 MAC을 Src MAC으로,
     다음 홉의 MAC을 Dst MAC으로 새로 캡슐화
       │ L2(Src=R1_out, Dst=R2_in) | L3(Dst=8.8.8.8) | DATA
       ▼
[Router R2] ...
       ▼
[Google DNS 8.8.8.8]
```

**왜 매 홉마다 MAC이 Rewrite 되는가?**
1. L3 장비(라우터)는 IP 헤더를 봐야 라우팅 결정 가능
2. 따라서 들어온 프레임의 L2 헤더를 **디캡슐화**한 후 L3 확인
3. 다음 홉으로 보낼 때 내보내는 인터페이스의 MAC을 Src로, 다음 홉의 MAC을 Dst로 **새로 캡슐화** 필요

---

## 4. 종합 시나리오: 192.168.0.10이 8.8.8.8과 통신할 때

```
[ Host A 192.168.0.10/24 ]
   │ Gateway 192.168.0.1
   │
   │ ① Subnet Mask로 비교 → 8.8.8.8은 외부 네트워크
   │ ② ARP로 Gateway(192.168.0.1)의 MAC 학습
   │ ③ 프레임 생성:
   │    L2: Src=HostA_MAC, Dst=GW_MAC
   │    L3: Src=192.168.0.10, Dst=8.8.8.8
   │
   ▼
[ Gateway / Router 192.168.0.1 ]
   │ ④ L2 디캡슐화
   │ ⑤ L3 확인: Dst=8.8.8.8 → 라우팅 테이블 조회
   │ ⑥ (사설망이라면) NAT로 Src IP 변환 (예: 203.0.113.5:xxxxx)
   │ ⑦ 다음 홉(ISP 라우터) MAC을 ARP로 학습
   │ ⑧ 새로운 L2 헤더로 재캡슐화
   │
   ▼
[ ISP Router → ... → Google 라우터 → 8.8.8.8 ]
```

- L3 IP는 (NAT 구간을 제외하면) 종단 간 그대로 유지
- L2 MAC은 매 라우터마다 새로 작성됨

---

## 관련 문서

- 통신 모델·캡슐화 흐름: [네트워크 기초 ① 통신 모델](/posts/network-01-osi-tcpip/)
- IP 주소 체계·서브넷 마스크·NAT: [네트워크 기초 ② IPv4 주소 체계와 서브네팅](/posts/network-02-ipv4-subnetting/)
