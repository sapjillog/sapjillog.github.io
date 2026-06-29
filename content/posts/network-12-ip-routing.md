---
title: "네트워크 라우팅 ① IP Routing 기본 원리 — 라우터의 패킷 전달 과정"
date: 2026-06-28
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "라우팅", "IP라우팅", "라우팅테이블", "CEF"]
summary: "스위치는 MAC으로 스위칭하고 라우터는 IP로 라우팅한다. 라우터가 패킷 하나를 받았을 때 거치는 처리 단계, 라우팅 테이블·재귀 조회, 매 홉 L2 헤더 Rewrite, 패킷 폐기 케이스까지 따라가며 정리한다. 서버/네트워크 지식 시리즈 12편(라우팅 1편)."
---

> **개요** — Switch는 MAC으로 *"스위칭"* 하지만 Router는 IP로 *"라우팅"* 한다. 라우터가 패킷 하나를 받았을 때 어떤 단계로 처리하는지를 송수신 호스트·중간 라우터 관점에서 따라간다.
>
> 사전 지식: [통신 모델 OSI·TCP/IP](/posts/network-01-osi-tcpip/) · [ARP·ICMP·Gateway](/posts/network-03-arp-icmp-gateway/)

---

## 1. Switch vs Router

| 항목 | Switch | Router |
| --- | --- | --- |
| 처리 계층 | L2 (Data Link) | **L3 (Network)** |
| 식별 주소 | MAC | **IP** |
| 사용 테이블 | MAC Address Table | **Routing Table** |
| 통신 범위 | LAN (단일 Broadcast Domain) | LAN/WAN (네트워크 간) |

**왜 MAC만 쓰지 않나?** 인터넷 전체에서 MAC만 쓰면 모든 라우터가 수십억 개 MAC을 학습해야 한다 — 불가능. 그래서 라우터는 **개별 호스트가 아닌 네트워크(서브넷) 단위**로 경로를 관리한다. "192.168.2.0/24는 R2에 있다" 한 줄로 200대를 처리.

---

## 2. Routing Table

### 2.1 가장 먼저 학습되는 경로 — Connected

인터페이스에 IP를 부여하는 순간 **자동으로** 라우팅 테이블에 등록된다.

```cisco
R1(config)# interface fa0/0
R1(config-if)# ip address 192.168.1.254 255.255.255.0
R1(config-if)# no shutdown
```

```cisco
R1# show ip route
C    192.168.1.0/24 is directly connected, FastEthernet0/0
L    192.168.1.254/32 is directly connected, FastEthernet0/0
```

### 2.2 학습 방식과 AD(Administrative Distance)

| 코드 | 학습 방식 | AD |
| --- | --- | --- |
| **C** (Connected) | IP 설정 시 자동 | 0 |
| **L** (Local) | 인터페이스 IP 자체(/32) | 0 |
| **S** (Static) | 관리자 수동 입력 | 1 |
| **D** (EIGRP) | 동적 | 90 |
| **O** (OSPF) | 동적 | 110 |
| **R** (RIP) | 동적 | 120 |

### 2.3 엔트리 해석

```
S     192.168.2.0/24 [1/0] via 192.168.12.2
│     │              │  │   └─ Next-Hop IP
│     │              │  └─ Metric (Static은 0)
│     │              └─ AD (Static = 1)
│     └─ 목적지 네트워크
└─ 학습 방식
```

---

## 3. 라우팅 동작 시나리오

H1(192.168.1.1) → H2(192.168.2.2), 중간에 R1·R2.

```
[H1 192.168.1.1] GW:.254
   │
[ R1 ]──── 192.168.12.0/24 ────[ R2 ]
 Fa0/0:.1.254  Gi0/2:.12.1   Gi0/2:.12.2  Fa1/0:.2.254
                                 │
                              [H2 192.168.2.2] GW:.254
```

### 3.1 H1의 의사결정

- **Q1. 목적지가 Local인가 External인가?** H1(192.168.1.1/24)에게 H2(192.168.2.2)는 외부 → **Default Gateway** 사용.
- **Q2. Gateway의 MAC을 아는가?** ARP 테이블에 있으면 사용, 없으면 ARP Request로 학습.

### 3.2 H1이 보내는 프레임

```
L2 [Src MAC: H1, Dst MAC: R1_Fa0/0_MAC]
L3 [Src IP : 192.168.1.1, Dst IP : 192.168.2.2]
```

> **L2 Dst MAC = R1 인터페이스 MAC**, **L3 Dst IP = 최종 목적지 H2 IP**. 이 구분이 라우팅 이해의 핵심.

### 3.3 R1의 처리 순서

```
1. FCS 검증 (틀리면 폐기)
2. Dst MAC이 자신/Broadcast/가입 Multicast면 진행
3. L2 Decapsulation → L3 헤더 확인
4. IP Checksum 검증
5. Dst IP(192.168.2.2) 확인
6. Routing Table 조회 → 일치 항목
7. TTL -1, Checksum 재계산
8. Next-Hop용 새 L2 프레임 구축 (Src=출력 인터페이스 MAC, Dst=Next-Hop MAC[ARP])
9. 출력 인터페이스로 전송
```

**재귀적 조회(Recursive Lookup)**: `192.168.2.2`→Next-Hop `192.168.12.2`→`192.168.12.0/24`는 Gi0/2에 직접 연결→Gi0/2로 전송.

### 3.4 매 홉 L2 Rewrite

> **⚠️ 핵심 규칙** — L3의 Src/Dst **IP는 종단 간 유지**(NAT 예외), L2의 Src/Dst **MAC은 매 홉마다 Rewrite**, **TTL은 매 홉 -1**.

| 구간 | Src MAC | Dst MAC | Src IP | Dst IP | TTL |
| --- | --- | --- | --- | --- | --- |
| H1→R1 | H1 | R1_Fa0/0 | .1.1 | .2.2 | 64 |
| R1→R2 | R1_Gi0/2 | R2_Gi0/2 | .1.1 | .2.2 | 63 |
| R2→H2 | R2_Fa1/0 | H2 | .1.1 | .2.2 | 62 |

R2는 `192.168.2.0/24`가 직접 연결돼 있어 별도 라우팅 없이 ARP 후 H2로 직접 전달.

---

## 4. 패킷이 폐기되는 케이스

| 폐기 조건 | 시점/결과 |
| --- | --- |
| FCS 부정확 | L2 수신 직후 |
| IP Checksum 부정확 | L3 처리 시 |
| **TTL = 0** | 폐기 + ICMP Time Exceeded |
| 목적지가 Routing Table에 없음 | Drop + ICMP Destination Unreachable |
| MTU 초과 + DF bit | Drop + ICMP Fragmentation Needed |

> **팁 — Traceroute 원리**: TTL을 1, 2, 3…으로 늘려 보내면 각 라우터가 TTL=0으로 만들고 ICMP Time Exceeded를 응답 → 경로상 라우터를 차례로 식별.

---

## 5. show 명령어

```cisco
R1# show ip route                     ! 전체 라우팅 테이블
R1# show ip route 192.168.2.2         ! 특정 IP의 최적 경로
R1# show ip route connected           ! Connected만
R1# show ip route static              ! Static만
R1# show ip arp                       ! ARP 테이블
R1# show ip interface brief           ! 인터페이스 IP 요약
R1# show ip cef                       ! CEF(FIB) 테이블
```

```cisco
R1# show ip route 192.168.2.2
Routing entry for 192.168.2.0/24
  Known via "static", distance 1, metric 0
  * 192.168.12.2          ← '*' 표시가 실제 선택되는 Next-Hop
```

---

## 6. CEF (Cisco Express Forwarding) — 참고

운영 라우터는 Routing Table을 매번 조회하지 않고 **CEF**로 최적화한다.

| 구조 | 역할 |
| --- | --- |
| **RIB** | 일반 Routing Table (`show ip route`) |
| **FIB** | RIB을 변환한 하드웨어 친화적 테이블 |
| **Adjacency Table** | Next-Hop MAC 등 L2 정보 |

CEF는 **하드웨어 가속(ASIC)**으로 Gbps급 전달.

---

## 7. 실무 빠른 점검

| 증상 | 점검 명령 |
| --- | --- |
| 패킷이 안 나감 | `show ip route X.X.X.X` (경로 존재?) |
| 잘못된 경로로 감 | `show ip route X.X.X.X` (어떤 항목 선택?) |
| ARP 안 됨 | `show ip arp X.X.X.X` |
| 인터페이스 down | `show ip interface brief` |

---

## 관련 문서

- 통신 모델·계층별 헤더: [통신 모델 OSI·TCP/IP](/posts/network-01-osi-tcpip/)
- ARP와 MAC Rewrite 메커니즘: [ARP·ICMP·Gateway](/posts/network-03-arp-icmp-gateway/)
- (다음 편) Static Route와 Floating Static
