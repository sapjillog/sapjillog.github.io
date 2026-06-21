---
title: "STP(Spanning Tree Protocol) 기본 동작 원리"
date: 2026-06-21
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "switching", "stp", "spanning-tree", "루프방지"]
summary: "스위치 이중화 시 발생하는 L2 루프의 문제점과 STP의 Root Bridge 선출, 포트 상태 천이 과정을 정리합니다."
---

> **개요** — 스위치 간 이중화는 가용성을 높이지만, 동시에 **L2 루프**라는 치명적 문제를 일으킨다.
> STP(IEEE 802.1D)는 물리적 이중화를 유지한 채로 논리적 루프를 차단하는 프로토콜이다.
> 본 문서는 루프가 왜 위험한지, STP가 어떻게 Root Bridge를 선출하고 포트를 막는지를 정리한다.

---

## 1. 왜 STP가 필요한가 - L2 루프의 비극

### 1.1 이중화의 부작용

```
[그림-1] 단일 링크
SW1 ──── SW2     ← 케이블 끊기면 통신 단절 (단일 장애 지점)

[그림-2] 이중화
SW1 ═══ SW2     ← 두 가닥, 가용성 ↑
                 ← 하지만 L2 루프 발생!
```

### 1.2 루프가 일으키는 일

H1이 ARP Request(Broadcast)를 보내면:

```
1. H1 → SW1
2. SW1: 두 케이블 모두로 SW2에 Flooding
3. SW2: 두 경로로 같은 프레임을 두 번 수신
4. SW2: 각각을 다른 케이블로 다시 SW1에 Flooding
5. SW1: 또 다시 두 케이블로... (무한 반복)
```

### 1.3 Ethernet에는 TTL이 없다

> **이더넷 프레임에는 TTL 필드가 없다.**
> L3 IP 패킷은 TTL이 0이 되면 폐기되지만, L2 프레임은 영원히 순환할 수 있다.

| 결과 |
| --- |
| **Broadcast Storm**: 브로드캐스트가 지수적으로 증가 → 네트워크 마비 |
| **MAC Table Instability**: 같은 MAC이 여러 포트에서 학습됨 → 정상 통신 불가 |
| **CPU/메모리 고갈**: 스위치 hang → 결국 한 장비가 죽어야 멈춤 |

### 1.4 STP의 해결책

> 물리적으로는 루프지만, **STP가 특정 포트를 BLK(Block)** 시켜 논리적 루프가 없는 토폴로지를 만든다.

```
물리 토폴로지:        STP 적용 후 (논리):
SW1 ═══ SW2           SW1 ──── SW2
                            BLK
                       ← 한 가닥은 차단
                       ← 장애 시 BLK가 풀려 자동 복구
```

---

## 2. STP 기본 개념

### 2.1 BPDU (Bridge Protocol Data Unit)

스위치가 STP 토폴로지를 형성하기 위해 주기적으로 교환하는 **L2 프레임**.

```
BPDU 주요 필드:
- Bridge ID  (Priority + MAC)
- Root Bridge ID
- Path Cost to Root
- Sender Port ID
- Timers (Hello, Max Age, Forward Delay)
```

| 항목 | 기본값 |
| --- | --- |
| Hello Interval | **2초** — BPDU 전송 주기 |
| Max Age | **20초** — BPDU 미수신 허용 시간 |
| Forward Delay | **15초** — Listening/Learning 각 단계 시간 |

### 2.2 Bridge ID

```
Bridge ID = [ Priority (16 bit) ] + [ MAC Address (48 bit) ]
```

| 필드 | 크기 | 설명 |
| --- | --- | --- |
| Priority | 16 bit | 기본값 **32768** (4096 배수로 변경 가능) |
| MAC | 48 bit | 스위치의 기본 MAC |

> **System ID Extension (PVST 환경)** — Cisco의 PVST는 Priority 필드를 12bit Priority + 4bit는 VLAN ID로 사용. 기본 Priority `32768 + VLAN ID(1)` = **32769**로 표시되는 이유.

### 2.3 Root Bridge

> 전체 스위치 중 **가장 좋은(=낮은) Bridge ID** 를 가진 장비.
> STP 토폴로지의 중심이 되며, 모든 경로 계산의 기준점.

비교 순서:
1. **Priority** 가 낮은 쪽 우선
2. Priority 같으면 **MAC**이 낮은 쪽 우선

### 2.4 포트 역할

| 역할 | 의미 | 동작 |
| --- | --- | --- |
| **Root Port (RP)** | Non-Root 스위치에서 Root까지 **가장 짧은 경로**의 포트 | Forwarding |
| **Designated Port (DP)** | 각 LAN(또는 링크)당 **가장 좋은** 포트 | Forwarding |
| **Non-Designated / Alternate** | 위에 해당하지 않는 포트 | **Blocking** |

**규칙**:
- Root Bridge의 모든 포트는 **항상 Designated**
- Non-Root 스위치는 RP를 1개씩 가짐
- 두 Non-Root 스위치 사이 링크에서 둘 중 더 좋은 BID를 가진 쪽이 DP, 다른 쪽이 BLK

### 2.5 Path Cost (대역폭 기반)

Root까지의 누적 비용. 낮을수록 좋다.

| 링크 속도 | STP Cost (Legacy) | RSTP Cost |
| --- | --- | --- |
| 10 Mbps | 100 | 2,000,000 |
| 100 Mbps | 19 | 200,000 |
| 1 Gbps | 4 | 20,000 |
| 10 Gbps | 2 | 2,000 |
| 100 Gbps | - | 200 |

---

## 3. STP 동작 순서

### 3.1 단계별 흐름

```
1. 모든 스위치가 BPDU 전송 (자신이 Root라고 가정)
2. 가장 좋은 BID를 가진 스위치가 Root Bridge로 결정
3. Non-Root 스위치는 Root Bridge로 가는 가장 짧은 경로의 포트를 Root Port로 지정
4. 각 링크에서 Designated Port 선출
5. 나머지 포트는 Blocking
```

### 3.2 시나리오 — 3대 스위치 삼각형

```
SW1 MAC: AAA, Priority 32768
SW2 MAC: BBB, Priority 32768
SW3 MAC: CCC, Priority 32768
모든 장비 동일 Priority
```

**Step 1**: BID 비교 → MAC이 가장 낮은 SW1이 Root Bridge

**Step 2**: SW2, SW3의 Root Port 결정
- SW2 → SW1로 가는 가장 짧은 경로의 포트 = Root Port
- SW3 → SW1로 가는 가장 짧은 경로의 포트 = Root Port

**Step 3**: Designated Port 결정
- SW1의 모든 포트 = Designated
- SW2 ↔ SW3 링크에서 BID 비교 → SW2(BBB) < SW3(CCC) → SW2 쪽이 Designated, SW3 쪽이 **Blocking**

```
SW1 (Root)
  ├── DP ── RP → SW2
  └── DP ── RP → SW3
                  │
                  DP ─ BLK ── SW2  (SW3의 BLK 포트)
```

---

## 4. 포트 상태 천이

```
[Disabled] → [Blocking] → [Listening] → [Learning] → [Forwarding]
              ↑              15 sec        15 sec
              └──────── 20 sec (Max Age)
```

| 상태 | 시간 | BPDU 송수신 | MAC 학습 | 데이터 전송 |
| --- | --- | --- | --- | --- |
| **Disabled** | - | X | X | X |
| **Blocking** | (Max Age 20s) | 수신만 | X | X |
| **Listening** | 15s | ✅ | X | X |
| **Learning** | 15s | ✅ | ✅ | X |
| **Forwarding** | 영구 | ✅ | ✅ | ✅ |

### 4.1 컨버전스 시간

- **Listening → Forwarding 까지**: 30초 (Listening 15s + Learning 15s)
- **Blocking → Forwarding 까지**: **50초** (Max Age 20s + Listening 15s + Learning 15s)

> **⚠️ 50초의 의미** — 운영 환경에서 STP 컨버전스에 50초가 걸리면 그 동안 통신 단절. 사용자 체감 큼. 이 문제 해결을 위한 RSTP/Portfast가 주요 튜닝 주제가 됨.

### 4.2 디버그로 천이 확인

```cisco
SW1# debug spanning-tree events
00:14:57: STP: VLAN0001 Fa0/1 -> listening
00:15:12: STP: VLAN0001 Fa0/1 -> learning
00:15:27: STP: VLAN0001 Fa0/1 -> forwarding
```

---

## 5. show 명령어 해석

### 5.1 전체 STP 상태

```cisco
SW1# show spanning-tree

VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769                ← Root Bridge Priority
             Address     000f.34ca.1000       ← Root MAC
             Cost        19                   ← SW1에서 Root까지 비용
             Port        19 (FastEthernet0/17)← Root로 가는 포트
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769                ← 자신의 Priority
             Address     0011.bb0b.3600       ← 자신의 MAC
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time 300

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ----
Fa0/14              Desg FWD 19        128.16   P2p
Fa0/17              Root FWD 19        128.19   P2p
```

### 5.2 해석 포인트

| 영역 | 의미 |
| --- | --- |
| Root ID | 토폴로지의 Root Bridge 정보 |
| Cost | 자신부터 Root까지의 누적 비용 |
| Port | Root로 향하는 자신의 Root Port |
| Bridge ID | 자신의 BID |
| Role | Desg(Designated) / Root / Altn(Alternate) |
| Sts | FWD(Forwarding) / BLK(Blocking) / LRN(Learning) |

### 5.3 자신이 Root인 경우

```cisco
SW3# show spanning-tree
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     000f.34ca.1000
             This bridge is the root          ← 자신이 Root
  ...
```

---

## 6. Root Bridge 수동 지정

> 자동 선출에 맡기면 가장 오래된(=낮은 MAC) 장비가 Root가 되는 경우가 많은데, 이는 성능·위치상 부적절할 수 있다. **운영에서는 코어 스위치를 Root로 명시 지정**한다.

### 6.1 방법 1: Priority 직접 변경

```cisco
SW1(config)# spanning-tree vlan 1 priority 4096       ! 4096의 배수만 가능
```

- Priority는 0, 4096, 8192, ... 등 4096의 배수만 입력 가능
- 가장 낮은 값을 가진 장비가 Root가 됨

### 6.2 방법 2: 자동화 매크로

```cisco
SW1(config)# spanning-tree vlan 1 root primary       ! Root로 지정
SW2(config)# spanning-tree vlan 1 root secondary     ! Backup Root 지정
```

내부적으로는 Priority를 조정 (primary는 28672 또는 더 낮게, secondary는 28672 + 4096).

### 6.3 VLAN별 다른 Root (Load Balancing)

```cisco
! Core SW1을 VLAN 10,20의 Root로
SW1(config)# spanning-tree vlan 10,20 priority 4096

! Core SW2를 VLAN 30,40의 Root로
SW2(config)# spanning-tree vlan 30,40 priority 4096
```

→ VLAN별로 트래픽 경로가 달라지면서 두 코어 스위치의 부하 분산 가능.

---

## 7. PVST (Per-VLAN Spanning Tree)

> Cisco의 기본 STP 동작. **VLAN별로 독립된 STP 인스턴스**를 운영한다.

### 7.1 IEEE 802.1D vs Cisco PVST

| 항목 | 802.1D | PVST |
| --- | --- | --- |
| 인스턴스 수 | 1개 (모든 VLAN 공유) | **VLAN별로 N개** |
| 표준 | IEEE | Cisco |
| Root 분리 | 불가 | **VLAN별 다른 Root 가능** |

### 7.2 PVST의 효율적 사용

```
VLAN 10 → Root는 SW1 (왼쪽 경로 사용, 오른쪽 BLK)
VLAN 20 → Root는 SW2 (오른쪽 경로 사용, 왼쪽 정지)
```

물리 토폴로지는 동일하지만 VLAN별로 다른 경로 사용 → 양쪽 링크 모두 활성화.

```cisco
SW1(config)# spanning-tree vlan 10 priority 4096    ! VLAN 10 Root = SW1
SW2(config)# spanning-tree vlan 20 priority 4096    ! VLAN 20 Root = SW2
```

### 7.3 PVST의 비효율적 운영

VLAN 수가 많은데 Root를 분산하지 않으면 한쪽 링크만 사용됨 → 다른 링크는 BLK로 낭비.

---

## 8. STP 운영 점검 체크리스트

| # | 항목 | 명령어 |
| --- | --- | --- |
| 1 | Root Bridge 위치 확인 | `show spanning-tree | include Root` |
| 2 | 자신의 BID 확인 | `show spanning-tree | include Bridge` |
| 3 | 각 포트의 역할 | `show spanning-tree interface fa0/X` |
| 4 | 토폴로지 변경 횟수 | `show spanning-tree detail | include changes` |
| 5 | 차단된 포트 위치 | `show spanning-tree blockedports` |
| 6 | 디버그 (긴급 시) | `debug spanning-tree events` |

### 8.1 비정상 시그널

| 증상 | 가능한 원인 |
| --- | --- |
| Topology Change 잦음 | 사용자 PC가 Up/Down 반복 → Portfast 미적용 |
| 예상 외 스위치가 Root | Priority 미설정 또는 MAC이 더 낮음 |
| 데이터가 BLK 포트로 흐름 | LoopGuard·BPDUGuard 트리거 없이 케이블 단방향 장애 |

---

## 9. 다음 학습 — STP의 한계 극복

STP의 **50초 컨버전스**는 현대 네트워크에 너무 느리다.
다음 노트에서 다룰 주제:

- **RSTP (Rapid STP)**: 컨버전스를 수 초로 단축
- **Portfast**: 호스트 연결 포트의 즉시 Forwarding
- **RootGuard / BPDUGuard / LoopGuard**: STP 토폴로지 보호

→ RSTP와 STP 튜닝

---

## 관련 문서

- Trunk 위에서 STP가 동작: [802.1Q Trunk와 Native VLAN](/posts/network-07-802-1q/)
- RSTP 및 STP 강화 기능: RSTP와 STP 튜닝
- 링크 묶기로 STP 단점 회피: EtherChannel (LACP/PAgP)
