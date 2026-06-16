---
title: "네트워크 스위칭 ① LAN과 스위치 — 충돌·브로드캐스트 도메인과 MAC 학습"
date: 2026-06-16
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "스위칭", "LAN", "스위치", "충돌도메인", "브로드캐스트도메인", "MAC학습"]
summary: "Hub의 한계에서 출발해 Switch의 충돌/브로드캐스트 도메인 분리, CSMA/CD, MAC 주소 학습(Flood & Learn), Port Security까지 L2 스위칭의 기초를 정리한다. 서버/네트워크 지식 시리즈 스위칭 1편."
---

> **개요** — 모든 LAN 통신의 시작은 Hub의 한계에서 비롯됐다. 이 글은 LAN의 종류, Hub와 Switch의 차이, CSMA/CD 충돌 제어, 그리고 스위치가 어떻게 MAC 주소를 학습해 프레임을 전달하는지를 정리한다.
>
> 사전 지식: [네트워크 기초 ① 통신 모델](/posts/network-01-osi-tcpip/) (OSI/TCP-IP 계층), [네트워크 기초 ③ ARP·ICMP·Gateway](/posts/network-03-arp-icmp-gateway/) (MAC·ARP)

---

## 1. LAN의 정의와 범위

**LAN(Local Area Network)** — 비교적 작은 단위로 구성되는 로컬 네트워크.
규모보다 *"로컬"* 이 핵심이다.

| 유형 | 설명 | 장비 예 |
| --- | --- | --- |
| **SOHO LAN** | 가정/소규모 사무실. 인터넷 연결·파일 공유 목적 | AP, 가정용 공유기, 소형 Switch |
| **Enterprise LAN** | 대규모 기업. 한 건물 또는 여러 건물의 수백~수천 대 연결 | Catalyst, Nexus 등 엔터프라이즈급 Switch |

LAN을 구성하는 대표 전송 장비는 **Switch**이며, 통신 프로토콜은 **Ethernet(802.3)**.

> 참고: LAN과 WAN의 구분, OSI/TCP-IP 계층 위치는 [네트워크 기초 ① 통신 모델](/posts/network-01-osi-tcpip/) 참고.

---

## 2. Hub의 한계 - Collision Domain

Hub는 **전기적 신호를 단순 복제(Repeat)** 하는 비지능형 장비다.

### 2.1 동작 방식

```
[H3] → Hub → [H1], [H2]에 동시 복제
- H1: 자신의 데이터 → 수신
- H2: 자신 데이터 아님 → 폐기
```

Hub의 모든 포트는 **동일한 전기 신호 경로**를 공유하므로 동시에 두 호스트가 전송하면 신호가 충돌한다.

### 2.2 Collision Domain

```
[H1] ─┐
[H2] ─┼─ Hub ─ [H3]
[H4] ─┘
```

- **Collision Domain**: 충돌이 발생할 가능성이 있는 전기적 영역
- Hub에 연결된 모든 단말은 같은 Collision Domain에 속함
- 동시에 데이터를 보내면 → **Collision(충돌)** 발생 → H3은 아무것도 수신하지 못함

### 2.3 CSMA/CD (Half-Duplex)

충돌 문제를 해결하기 위한 프로토콜.

| 약어 | 의미 |
| --- | --- |
| **CS** (Carrier Sense) | 다른 호스트가 전송 중인지 회선을 *"청취"* |
| **MA** (Multiple Access) | 회선이 비어 있으면 누구나 사용 가능 |
| **CD** (Collision Detection) | 두 호스트가 동시 전송하면 충돌 감지 후 재전송 |

> **⚠️ 한계**
> - 회선이 비어 있을 때만 전송 가능 → **Half-Duplex** (한 번에 한 방향)
> - 장비가 많을수록 충돌 빈도 ↑ → 대역폭 낭비

---

## 3. Switch - 지능형 LAN 장비

Switch는 **각 포트가 독립된 Collision Domain**을 가진다. 즉, Hub의 충돌 문제를 근본적으로 해결.

| 특성 | Hub | Switch |
| --- | --- | --- |
| 동작 계층 | L1 (Physical) | **L2 (Data Link)** |
| 신호 처리 | 단순 복제(Repeat) | MAC 학습 후 선택 전송 |
| Collision Domain | 모든 포트 공유 | **포트별 분리** |
| Duplex | Half-Duplex | **Full-Duplex** |
| 동시 송수신 | 불가 | 가능 |
| 충돌 발생 | 빈번 | 사실상 없음 |

> **Full-Duplex의 의미** — Switch는 각 포트별로 송신·수신 회선이 분리되어 있어, 모든 호스트가 **동시에 송수신 가능**하다. CSMA/CD가 무의미해진다.

### 3.1 Switch 하단에 Hub를 두는 경우

```
Switch ─ Hub ─ [H1]
              [H2]
              [H3]
```

- Hub 하단의 H1/H2/H3는 다시 **하나의 Collision Domain**에 속함
- 운영 환경에서는 가급적 Hub 사용 지양

---

## 4. Broadcast Domain

> 브로드캐스트는 수신자 동의와 관계 없이 "everyone receives".

### 4.1 Switch와 Broadcast

- Switch는 브로드캐스트 프레임을 **수신 포트를 제외한 모든 포트로 Flooding**
- 브로드캐스트의 Destination MAC: `FF:FF:FF:FF:FF:FF`
- 대표적 브로드캐스트: **ARP Request**

### 4.2 Broadcast Domain의 정의

```
[H1] ┐
[H2] ┤
[H3] ├── Switch ──── 같은 Broadcast Domain
[H4] ┤
[H5] ┘
```

**브로드캐스트 도메인** = 브로드캐스트 트래픽을 수신하는 네트워크 디바이스 모음

### 4.3 Broadcast Domain의 문제와 분리

- 너무 많은 브로드캐스트 → **대역폭과 CPU 리소스 낭비** (브로드캐스트 스톰)
- 하드웨어 발전으로 도메인당 약 1,000대까지는 무리 없이 운용 가능하지만 **분리가 원칙**

**분리 방법**:

| 방법 | 장비 | 특징 |
| --- | --- | --- |
| **Router로 분리** | L3 Router | 물리적으로 서로 다른 네트워크 |
| **VLAN으로 분리** | L2/L3 Switch | 논리적 분리 (다음 편: VLAN 개념과 구성) |

---

## 5. Switch의 MAC Address 학습 (Flood & Learn)

Switch는 OSI L2에서 동작하며, **목적지 MAC**으로 전달 포트를 결정한다.
이를 위해 **MAC Address Table** (= CAM Table)을 유지한다.

### 5.1 학습 시나리오 (1) — 첫 통신, ARP 사용

H1(MAC `AAA`)이 H2(MAC `BBB`)에게 처음 통신.
H1은 H2의 MAC을 모르므로 **ARP Request(Broadcast)** 부터 보낸다.

```
[Step 1] H1 → Switch
   Src MAC: AAA, Dst MAC: FFF (Broadcast)
   "192.168.1.2의 MAC 알려줘"
```

**Switch 동작**:
1. 수신 프레임의 Src MAC(`AAA`)을 MAC Table에 기록 (수신 포트 번호와 함께)
2. Dst MAC이 Broadcast → **수신 포트를 제외한 모든 포트로 Flooding**

```
MAC Table
| MAC | Port |
| AAA | 1    |
```

```
[Step 2] H2 → Switch (ARP Reply, Unicast)
   Src MAC: BBB, Dst MAC: AAA
   "내 MAC은 BBB"
```

**Switch 동작**:
1. Src MAC(`BBB`)을 MAC Table에 추가
2. Dst MAC(`AAA`)이 MAC Table에 있음 → 해당 포트로만 직접 전달 (Flooding 아님)

```
MAC Table
| MAC | Port |
| AAA | 1    |
| BBB | 2    |
```

### 5.2 학습 시나리오 (2) — MAC을 이미 알지만 Switch는 모를 때

H1이 H2의 MAC을 ARP Cache로 이미 알고 있어도, Switch가 처음 보는 트래픽이면 같은 학습 과정이 일어난다.

```
[Step 1] H1 → Switch (일반 Unicast)
   Src MAC: AAA, Dst MAC: BBB

Switch 동작:
1. Src MAC(AAA) 학습
2. Dst MAC(BBB) — Table에 없음 → 수신 포트 제외 모든 포트로 Flooding (Unknown Unicast Flooding)
```

```
[Step 2] H2 응답 → Switch
   Src MAC: BBB, Dst MAC: AAA

Switch 동작:
1. Src MAC(BBB) 학습 → Table 완성
2. Dst MAC(AAA)이 Table에 있음 → 해당 포트로만 전달
```

> **핵심 원칙**
> - **Source MAC** → 항상 학습
> - **Destination MAC**
>   - Table에 **있음** → 해당 포트로 직접 전송 (Forwarding)
>   - Table에 **없음** → 수신 포트 제외 모든 포트로 Flooding (Unknown Unicast)
>   - Broadcast(`FF:FF:FF:FF:FF:FF`) → 항상 Flooding

### 5.3 MAC Table 확인

```cisco
SW1#show mac address-table
Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0011.bb0b.3601    DYNAMIC     Fa0/1
   1    0019.569d.5702    DYNAMIC     Fa0/2
```

- **Dynamic 항목**의 기본 Aging Time: **300초** (5분)
- 그 시간 동안 해당 MAC에서 프레임이 오지 않으면 Table에서 삭제

```cisco
! Aging Time 변경
SW1(config)# mac address-table aging-time 600

! MAC Table 초기화
SW1# clear mac address-table dynamic

! 정적(Static) MAC 등록 — 보안·고정 매핑용
SW1(config)# mac address-table static 0011.2233.4455 vlan 10 interface fa0/5
```

---

## 6. 도메인 정리표

| 도메인 | 분리 단위 | 분리 방법 |
| --- | --- | --- |
| **Collision Domain** | Switch의 **포트** 또는 Full-Duplex 링크 | Hub → Switch 교체 |
| **Broadcast Domain** | **VLAN** 또는 **Router 인터페이스** | VLAN 분리, L3 라우팅 |

```
[Hub 환경]
─ 1 Collision Domain
─ 1 Broadcast Domain

[Switch 환경 (단일 VLAN)]
─ 포트마다 Collision Domain (사실상 0)
─ 1 Broadcast Domain

[Switch + VLAN 분리]
─ 포트마다 Collision Domain
─ VLAN당 Broadcast Domain
```

---

## 7. 실무 보강 - 보안과 운영

### 7.1 Unknown Unicast Flooding의 위험

- 정상적이지만 **트래픽 누출 위험**: 의도치 않은 포트에 데이터가 새어 나감
- 운영망에서 **잦은 MAC Table 갱신**이 발생하면 의심해 볼 만한 시그널 (스푸핑, MAC Flapping)

### 7.2 MAC Flapping

같은 MAC이 짧은 시간 안에 서로 다른 포트에서 학습되는 현상.
- 일반적 원인: **STP 루프**, **호스트 이동**, **이중 NIC 잘못된 구성**
- 로그 예: `%SW_MATM-4-MACFLAP_NOTIF: Host xxxx.xxxx.xxxx in vlan X is flapping between port FaX/X and FaX/X`

### 7.3 Port Security (스위치 포트 보안)

특정 포트에 학습 가능한 MAC 수를 제한하거나 정해진 MAC만 허용.

```cisco
SW1(config)# interface fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation shutdown
```

| 옵션 | 의미 |
| --- | --- |
| `maximum N` | 학습 가능한 MAC 수 제한 |
| `mac-address sticky` | 학습한 MAC을 자동으로 running-config에 등록 |
| `violation shutdown` | 위반 시 포트 err-disable (운영에서 가장 보수적) |
| `violation restrict` | 위반 프레임 폐기 + 로그/SNMP 알림 |
| `violation protect` | 위반 프레임 폐기 (로그 없음) |

---

## 8. 빠른 점검 명령어

```cisco
SW1# show mac address-table              ! 전체 MAC 테이블
SW1# show mac address-table count        ! 학습된 MAC 수
SW1# show mac address-table dynamic
SW1# show mac address-table interface fa0/1
SW1# show interfaces status              ! 포트 상태
SW1# show interfaces counters errors     ! 에러 카운터 (CRC, Collision)
SW1# show port-security                  ! Port Security 상태
```

---

## 9. 한 줄 요약

> **Hub는 충돌을, Switch는 학습을 한다.** Switch는 포트마다 Collision Domain을 나누고, MAC 학습(Flood & Learn)으로 트래픽을 정확한 포트에만 전달한다. 남은 과제는 Broadcast Domain 분리 — 그 답이 VLAN이다.

---

## 관련 문서

- 통신 모델·캡슐화 흐름: [네트워크 기초 ① 통신 모델](/posts/network-01-osi-tcpip/)
- ARP·MAC 학습이 결합되는 과정: [네트워크 기초 ③ ARP·ICMP·Gateway](/posts/network-03-arp-icmp-gateway/)
- Broadcast Domain 분리의 핵심 (다음 편): VLAN 개념과 구성
