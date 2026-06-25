---
title: "네트워크 스위칭 ⑥ RSTP와 STP 튜닝 — Portfast·RootGuard·BPDUGuard·LoopGuard"
date: 2026-06-25
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "스위칭", "RSTP", "STP", "Portfast", "BPDUGuard"]
summary: "STP의 50초 컨버전스를 수 초로 줄이는 RSTP와, Portfast·RootGuard·BPDUGuard·LoopGuard·UDLD로 토폴로지를 지키는 STP 강화 기능을 정리한다. 서버/네트워크 지식 시리즈 10편."
---

> **개요** — 기존 STP의 **50초 컨버전스**는 운영 환경에 너무 느리다. RSTP(IEEE 802.1w)는 이를 수 초로 단축하며, 다양한 STP 보호 기능(Portfast·RootGuard·BPDUGuard·LoopGuard)으로 토폴로지를 안정화할 수 있다.
>
> 사전 지식: [네트워크 스위칭 ⑤ STP 기본 동작](/posts/network-09-stp/) (Root Bridge, BID, BPDU, 포트 역할/상태)

---

## 1. STP의 한계와 RSTP의 등장

| 항목 | STP (802.1D) | RSTP (802.1w) |
| --- | --- | --- |
| 컨버전스 | **50초** | **수 초 (1~6s)** |
| 포트 상태 | Disabled/Blocking/Listening/Learning/Forwarding | **Discarding/Learning/Forwarding** |
| Listening 상태 | 존재 | 없음 (Discarding으로 통합) |
| 포트 역할 | Root/Designated/Non-Designated | Root/Designated/**Alternate/Backup** |
| BPDU 활용 | Root만 BPDU 생성 | **모든 스위치가 BPDU 생성** (Keepalive 역할) |
| 토폴로지 변경 알림 | TCN 별도 메커니즘 | BPDU 자체에 포함 |

> EIGRP/OSPF 같은 라우팅 프로토콜의 컨버전스는 1~5초인데, STP가 50초라 전체 네트워크 회복이 STP에 발목 잡힘 → RSTP 개발 동기.

---

## 2. RSTP 포트 상태 (3가지)

| 상태 | 의미 | MAC 학습 | 데이터 전송 |
| --- | --- | --- | --- |
| **Discarding** | STP의 Blocking + Listening 통합 | ❌ | ❌ |
| **Learning** | MAC 학습 | ✅ | ❌ |
| **Forwarding** | 정상 데이터 전송 | ✅ | ✅ |

```
STP:   Blocking → Listening → Learning → Forwarding (50초)
RSTP:  Discarding → Learning → Forwarding (수 초)
```

---

## 3. RSTP 포트 역할 (4가지)

| 역할 | 설명 |
| --- | --- |
| **Root Port (RP)** | Root Bridge까지 가장 짧은 경로의 포트 (Forwarding) |
| **Designated Port (DP)** | 각 LAN의 대표 포트 (Forwarding) |
| **Alternate Port** | RP의 백업. Discarding 상태이지만 즉시 RP로 승격 가능 |
| **Backup Port** | 같은 스위치 내 DP의 백업 (드물게 발생) |

> Alternate Port의 존재 덕분에 Root 경로 장애 시 즉시 우회 가능 → 빠른 컨버전스.

---

## 4. RPVST (Rapid PVST) 구성

> Cisco의 PVST를 RSTP 기반으로 동작시키는 것이 **Rapid PVST**.

```cisco
SW1(config)# spanning-tree mode rapid-pvst
SW2(config)# spanning-tree mode rapid-pvst
SW3(config)# spanning-tree mode rapid-pvst
```

확인:

```cisco
SW1# show spanning-tree
VLAN0001
  Spanning tree enabled protocol rstp           ← rstp (= RSTP)
  Root ID    Priority    4097
             Address     0011.bb0b.3600
             This bridge is the root
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------
Fa0/14              Desg FWD 19        128.16   P2p
Fa0/17              Desg FWD 19        128.19   P2p
```

> **참고 — STP→RSTP 전환**
> RPVST 모드 변경 명령 한 줄로 즉시 전환된다. 하위 호환(802.1D 스위치와 연동 가능)이지만, 양쪽이 모두 RSTP일 때 가장 빠른 컨버전스를 달성한다.

---

## 5. STP 튜닝 1 — Portfast

PC 부팅 시 Access 포트가 Listening(15s)→Learning(15s)을 거쳐 **30초간 통신 불가** → DHCP·PXE 부팅 실패 가능. **Portfast는 Listening/Learning을 생략하고 즉시 Forwarding**으로 진입한다(TCN도 미생성).

```cisco
! 특정 인터페이스
SW1(config-if)# spanning-tree portfast

! 모든 Access 포트 일괄 (전역)
SW1(config)# spanning-tree portfast default
```

> **⚠️ 위험 — 스위치/Hub에 Portfast 금지**
> Portfast 포트가 다른 스위치/허브와 연결되면 **즉시 루프 발생.** 반드시 **단일 호스트(PC/Server/Printer)** 포트에만 사용하고, 안전장치로 **BPDU Guard와 함께** 쓴다.

---

## 6. STP 튜닝 2 — RootGuard

의도하지 않은 스위치가 Root Bridge로 선출되는 것을 방지. RootGuard 포트가 **현재 Root보다 좋은 BPDU**를 수신하면 → **root-inconsistent**로 차단(정상 BPDU 복귀 시 자동 복구).

```cisco
SW2(config-if)# spanning-tree guard root
```

- 적용 위치: **Root로 연결되지 않는 방향**의 Designated 포트. ("이 포트에선 Root BPDU가 와선 안 된다"는 위치)

---

## 7. STP 튜닝 3 — BPDU Guard

호스트 연결 포트(Portfast)에서 **BPDU가 들어오면** 의도치 않은 스위치 연결 가능성 → 포트를 **err-disable(셧다운)**.

```cisco
! 인터페이스
SW2(config-if)# spanning-tree bpduguard enable

! Portfast 전역 포트에 자동 적용 (권장)
SW1(config)# spanning-tree portfast bpduguard default
```

복구:

```cisco
SW2(config-if)# shutdown
SW2(config-if)# no shutdown
! 또는 자동 복구
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300       ! 5분 후 재시도
```

### RootGuard vs BPDUGuard

| 항목 | RootGuard | BPDUGuard |
| --- | --- | --- |
| 트리거 | **Superior BPDU** 수신 | **모든 BPDU** 수신 |
| 적용 위치 | Trunk 포트 (Root 아닌 방향) | **Portfast 포트 (호스트)** |
| 결과 | root-inconsistent (자동 복구) | **err-disable** (수동 복구) |

---

## 8. STP 튜닝 4 — BPDU Filter

인터페이스에서 BPDU 송수신을 막음.

| 적용 위치 | 동작 |
| --- | --- |
| **전역** | Portfast 포트에서 동작. BPDU 수신 시 Portfast/Filter 해제(안전장치 있음) |
| **인터페이스** | 해당 포트 BPDU 완전 중지 (위험: 스위치 연결 시 루프) |

```cisco
SW1(config)# spanning-tree portfast bpdufilter default
```

> **⚠️ 위험 — 인터페이스 BPDU Filter**
> 스위치를 다른 스위치에 연결한 상태에서 적용하면 STP가 완전히 침묵 → 루프. 호스트 직접 연결일 때만 신중하게.

---

## 9. STP 튜닝 5 — LoopGuard

케이블 단방향 장애나 H/W 오동작으로 **BPDU를 정상 수신하지 못할 때**, 포트가 Forwarding으로 잘못 전환돼 루프 만드는 것을 방지(→ **loop-inconsistent** 차단).

```cisco
! 전역
SW3(config)# spanning-tree loopguard default
! 인터페이스
SW3(config-if)# spanning-tree guard loop
```

### LoopGuard vs RootGuard

| 항목 | LoopGuard | RootGuard |
| --- | --- | --- |
| 적용 포트 | **Root / Alternate Port** | Designated Port |
| 트리거 | BPDU 미수신 (단방향) | Superior BPDU 수신 |
| 보호 대상 | 단방향 케이블 장애 → 루프 | 임의 스위치의 Root 탈취 |

> 두 기능은 같은 포트에 동시 적용하지 않는다(적용 포트가 다름).

---

## 10. UDLD — 단방향 링크 감지

LoopGuard의 보조. 광케이블 등에서 **한쪽 방향만 끊긴 상태**를 감지.

```cisco
SW1(config)# udld enable           ! Normal mode
SW1(config)# udld aggressive       ! Aggressive (강제 err-disable)
SW1(config-if)# udld port aggressive
```

---

## 11. STP 강화 종합 적용 예시

```cisco
! ─── 전역 ───
SW1(config)# spanning-tree mode rapid-pvst
SW1(config)# spanning-tree portfast default
SW1(config)# spanning-tree portfast bpduguard default
SW1(config)# spanning-tree loopguard default
SW1(config)# spanning-tree vlan 1-4094 priority 4096      ! 코어를 Root로
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300

! ─── 사용자 포트 (Access) ───
SW1(config)# interface range fa0/1 - 20
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# spanning-tree portfast
SW1(config-if-range)# spanning-tree bpduguard enable

! ─── Trunk 포트 (다른 스위치 연결) ───
SW1(config)# interface gi0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# spanning-tree guard root
SW1(config-if)# udld port aggressive
```

---

## 12. STP 강화 기능 요약표

| 기능 | 목적 | 적용 위치 | 트리거 | 결과 |
| --- | --- | --- | --- | --- |
| **Portfast** | 호스트 포트 즉시 Forwarding | Access (호스트) | 인터페이스 Up | Listening/Learning 생략 |
| **BPDU Guard** | 사용자 측 BPDU 차단 | Portfast 포트 | BPDU 수신 | err-disable |
| **BPDU Filter** | BPDU 송수신 차단 | Portfast/특수 | - | BPDU 침묵 |
| **Root Guard** | 임의 Root 차단 | Designated 포트 | Superior BPDU | root-inconsistent |
| **Loop Guard** | 단방향 장애 보호 | Root/Alternate 포트 | BPDU 미수신 | loop-inconsistent |
| **UDLD** | 단방향 링크 감지 | 광 링크 | 단방향 트래픽 | err-disable(aggressive) |

---

## 13. MSTP — 다중 인스턴스 STP (참고)

PVST는 VLAN 수만큼 STP 인스턴스를 운영해 부담이 크다. MSTP(802.1s)는 여러 VLAN을 **인스턴스로 그룹화**해 효율을 개선한다. 대규모 환경 표준.

```cisco
SW1(config)# spanning-tree mode mst
SW1(config)# spanning-tree mst configuration
SW1(config-mst)# name REGION1
SW1(config-mst)# revision 1
SW1(config-mst)# instance 1 vlan 10,20,30
SW1(config-mst)# instance 2 vlan 40,50,60
```

---

## 14. 점검 명령어

```cisco
SW1# show spanning-tree summary
SW1# show spanning-tree inconsistentports      ! Root/Loop Inconsistent 포트
SW1# show errdisable recovery
SW1# show interfaces status err-disabled
SW1# show udld neighbors
```

---

## 관련 문서

- STP 기본 동작과 BID/포트 역할: [네트워크 스위칭 ⑤ STP 기본 동작](/posts/network-09-stp/)
- (다음 편) EtherChannel로 STP 자체를 회피
