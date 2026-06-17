---
title: "네트워크 스위칭 ② VLAN — 가상 LAN 개념과 Cisco IOS 구성"
date: 2026-06-17
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "스위칭", "VLAN", "브로드캐스트도메인", "access-port", "cisco-ios"]
summary: "하나의 물리 스위치를 논리적으로 여러 LAN처럼 나누는 VLAN. 왜 필요한지, VLAN의 종류, Cisco IOS 구성·검증, vlan.dat 백업과 보안 권고까지 정리한다. 서버/네트워크 지식 시리즈 스위칭 2편."
---

> **개요** — VLAN(Virtual LAN)은 **하나의 물리 스위치를 논리적으로 여러 개의 LAN처럼 분리**하는 기술이다. 이 글은 VLAN이 왜 필요한지, 무엇을 해결하는지, 그리고 Cisco IOS에서 어떻게 구성·확인하는지를 정리한다.
>
> 사전 지식: [네트워크 스위칭 ① LAN과 스위치](/posts/network-05-lan-switch-domains/) (충돌·브로드캐스트 도메인, MAC 학습)

---

## 1. 왜 VLAN이 필요한가

### 1.1 물리적 그룹화의 한계

부서별로 자체 스위치를 운영하는 회사를 상상해보자.

```
[ Computers 부서 ] ── SW-A
[ Help Desk     ] ── SW-B
[ Servers       ] ── SW-C
[ Human Resource] ── SW-D
                          ↓ (모두 한 코어 SW로 연결)
                       [ Core SW ]
```

이 디자인의 문제:

| # | 문제 |
| --- | --- |
| 1 | **Broadcast가 전사로 전파됨** — 한 PC가 보낸 ARP가 전 부서 트래픽으로 확산 |
| 2 | **부서 단위 격리가 불가능** — Help Desk SW 장애 시 HR이 격리되거나 전체 마비 |
| 3 | **부서 간 보안 적용이 어려움** — MAC Filtering으로 가능하지만 관리 부담 ↑ |
| 4 | **물리적 이동이 곧 논리적 이동** — 자리 옮기면 케이블·VLAN 모두 재설계 |

### 1.2 VLAN의 해결책

**같은 물리 스위치에 연결돼 있어도, 다른 VLAN이면 서로 통신할 수 없다.**

```
물리적으로는 같은 스위치 1대 안에 있지만,
논리적으로는 VLAN 10 / VLAN 20 / VLAN 30이 각각 독립된 LAN처럼 동작.
```

| 이점 | 설명 |
| --- | --- |
| **Broadcast Domain 분리** | VLAN별로 Broadcast가 격리됨 |
| **물리 ≠ 논리** | 어디에 꽂아도 같은 VLAN이면 같은 LAN |
| **보안 강화** | VLAN 간 통신은 L3 라우팅 필요 → ACL/방화벽 적용 가능 |
| **장애 격리** | 한 VLAN의 브로드캐스트 스톰이 다른 VLAN에 전파되지 않음 |
| **유연성** | VLAN 추가·이동이 케이블 작업 없이 명령어로 가능 |

> **핵심 규칙**
> - 동일 VLAN 내부: **L2 Switch**로 통신 (MAC 기반)
> - 서로 다른 VLAN: **L3 장비(Router 또는 L3 Switch)** 가 있어야 통신 가능 — Inter-VLAN Routing

---

## 2. VLAN의 종류

| VLAN | 범위 | 용도 |
| --- | --- | --- |
| **Default VLAN** | 1 | 모든 포트 기본 소속 |
| **Data VLAN** | 2 ~ 1001 | 일반 사용자 트래픽 |
| **Voice VLAN** | 별도 지정 | IP Phone 트래픽 분리 — 802.1Q Trunk와 Native VLAN |
| **Native VLAN** | Trunk 한정 | Trunk 링크의 태그 없는 트래픽 — 802.1Q Trunk와 Native VLAN |
| **Management VLAN** | 보통 1 (변경 권장) | 스위치 자체 관리(SSH/Telnet/SNMP) |
| **예약** | 1002 ~ 1005 | Token Ring/FDDI 호환용 (실사용 X) |
| **Extended VLAN** | 1006 ~ 4094 | 대규모 환경에서 추가 사용 |

> **⚠️ 주의 — Default VLAN 1을 그대로 쓰지 말 것**
> 모든 미할당 포트가 자동으로 속하므로 **보안적으로 취약**.
> 실무에서는 Management VLAN을 별도 번호로 분리하고, VLAN 1은 사용자 포트에서 제거하는 것이 좋다.

---

## 3. VLAN 구성 흐름

```
1) VLAN 정의 (번호와 이름)
2) 포트에 VLAN 할당 (Access 모드)
3) 검증 (show vlan, show interfaces switchport)
```

### 3.1 기본 상태 확인

```cisco
SW1# show vlan brief

VLAN Name                  Status    Ports
---- --------------------- --------- -------------------------------
1    default               active    Fa0/1, Fa0/2, Fa0/3, ... Gi0/2
1002 fddi-default          act/unsup
1003 token-ring-default    act/unsup
1004 fddinet-default       act/unsup
1005 trnet-default         act/unsup
```

처음에는 모든 포트가 VLAN 1(default)에 속해 있다.

### 3.2 VLAN 생성

```cisco
SW1(config)# vlan 50
SW1(config-vlan)# name Computers
SW1(config-vlan)# exit

SW1(config)# vlan 60
SW1(config-vlan)# name Servers
SW1(config-vlan)# exit
```

> VLAN 이름은 운영상 식별을 위해 항상 지정하는 습관을 들이는 게 좋다.

### 3.3 포트에 VLAN 할당 (Access 모드)

```cisco
SW1(config)# interface fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 50
SW1(config-if)# exit

SW1(config)# interface fa0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 50
```

**여러 포트를 한 번에 할당** — `interface range` 사용:

```cisco
SW1(config)# interface range fa0/1 - 10
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 50
```

### 3.4 결과 확인

```cisco
SW1# show vlan brief

VLAN Name                  Status    Ports
---- --------------------- --------- -------------------------------
1    default               active    Fa0/3, Fa0/4, Fa0/5, ...
50   Computers             active    Fa0/1, Fa0/2
60   Servers               active
```

```cisco
SW1# show interfaces fa0/1 switchport
Name: Fa0/1
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Access Mode VLAN: 50 (Computers)
Trunking Native Mode VLAN: 1 (default)
```

---

## 4. Access Port vs Trunk Port

| 구분 | Access Port | Trunk Port |
| --- | --- | --- |
| **목적** | 단일 단말 연결 | 스위치 간 다중 VLAN 전송 |
| **VLAN 수** | 1개 (+ Voice VLAN) | 다수 (1~4094) |
| **프레임** | 일반 Ethernet | **802.1Q 태그**가 붙은 프레임 |
| **명령** | `switchport mode access` | `switchport mode trunk` |
| **할당** | `switchport access vlan N` | `switchport trunk allowed vlan` |

> Trunk 상세는 다음 편(802.1Q Trunk와 Native VLAN)에서 다룬다.

---

## 5. VLAN 통신 실험 - 격리 확인

### 5.1 같은 VLAN 내 통신

```
H1 (Fa0/1, VLAN 50, 192.168.1.1)
H2 (Fa0/2, VLAN 50, 192.168.1.2)

H1> ping 192.168.1.2  ← 성공 (같은 VLAN, L2 통신)
```

### 5.2 다른 VLAN 간 통신

```
H1 (Fa0/1, VLAN 50, 192.168.1.1)
H3 (Fa0/3, VLAN 60, 192.168.1.3)
```

같은 서브넷이라도 VLAN이 다르면 통신 **불가**.
스위치는 H1의 ARP Request(Broadcast)를 VLAN 50 안으로만 Flooding하며, VLAN 60의 H3은 듣지 못한다.

→ Inter-VLAN Routing이 필요. (Router-on-a-Stick 또는 L3 Switch의 SVI)

---

## 6. VLAN 운영 명령어 정리

### 6.1 VLAN 생성·삭제·변경

```cisco
! 생성
SW1(config)# vlan 100
SW1(config-vlan)# name DEV

! 이름 변경
SW1(config)# vlan 100
SW1(config-vlan)# name DEVELOPMENT

! 비활성화 (포트는 유지, 트래픽 차단)
SW1(config)# vlan 100
SW1(config-vlan)# shutdown

! 삭제
SW1(config)# no vlan 100
```

> **⚠️ 위험 — 삭제 시 주의**
> 사용 중인 VLAN을 삭제하면 해당 VLAN에 속한 포트는 **inactive** 상태가 된다.
> 삭제 전 `show vlan brief`로 어떤 포트들이 영향받는지 반드시 확인.

### 6.2 포트 VLAN 변경

```cisco
! 다른 VLAN으로 이동
SW1(config-if)# switchport access vlan 200

! VLAN 1(default)로 복귀
SW1(config-if)# no switchport access vlan
```

### 6.3 VLAN.dat 파일

- VLAN 정보는 `running-config`가 아닌 별도 파일 `flash:vlan.dat`에 저장됨
- 따라서 **`copy running-config startup-config`만으로는 VLAN 정보가 백업되지 않음** — `vlan.dat`을 별도로 백업해야 완전한 복원 가능
- 공장 초기화 시:

```cisco
SW1# erase startup-config
SW1# delete flash:vlan.dat
SW1# reload
```

### 6.4 점검 명령어

```cisco
SW1# show vlan                          ! VLAN 상세
SW1# show vlan brief                    ! 요약
SW1# show vlan id 50                    ! 특정 VLAN
SW1# show interfaces switchport         ! 모든 포트 모드
SW1# show interfaces fa0/1 switchport   ! 단일 포트
SW1# show mac address-table vlan 50     ! 특정 VLAN의 MAC Table
```

---

## 7. IP 주소 설계와 VLAN

> 한 VLAN = 한 Broadcast Domain = 한 서브넷이 원칙.

| VLAN | 용도 | 서브넷 예시 | Gateway |
| --- | --- | --- | --- |
| 10 | Office | 192.168.10.0/24 | 192.168.10.254 |
| 20 | Server | 192.168.20.0/24 | 192.168.20.254 |
| 30 | Voice | 192.168.30.0/24 | 192.168.30.254 |
| 99 | Management | 192.168.99.0/24 | 192.168.99.254 |

> **💡 팁 — VLAN 번호 ↔ 3옥텟 매칭 관례**
> VLAN 10 → 192.168.**10**.0/24, VLAN 20 → 192.168.**20**.0/24 식으로 일치시키면 운영자가 IP만 봐도 어느 VLAN인지 즉시 파악할 수 있다.

---

## 8. 자주 발생하는 실수

| 증상 | 원인 |
| --- | --- |
| 포트에 VLAN 할당했는데 통신 안 됨 | `switchport mode access`를 빠뜨림 (negotiate 상태로 남음) |
| VLAN 삭제했더니 포트가 죽음 | 삭제 전 포트 재할당 필요 |
| `show vlan`에 Trunk 포트가 안 보임 | 정상. Trunk는 `show interfaces trunk`로 확인 |
| 다른 VLAN과 통신 안 됨 | L3 장비 부재. Inter-VLAN Routing 필요 |
| VLAN 정보가 reload 후 사라짐 | `vlan.dat`이 손상되었거나 VTP Client 상태 |

---

## 9. 보안 권고

1. **Default VLAN 1 사용 지양** — 모든 포트가 자동 소속되는 VLAN
2. **미사용 포트는 별도 VLAN(예: 999, "PARKING")에 격리** 후 `shutdown`
3. **Management VLAN을 사용자 VLAN과 분리**
4. **포트별로 명시적인 `switchport mode access` 선언** — DTP(Dynamic Trunking Protocol) 협상 비활성화
5. **VTP는 운영에서 가급적 Transparent 또는 Off**

미사용 포트 격리 예시:

```cisco
SW1(config)# vlan 999
SW1(config-vlan)# name PARKING
SW1(config-vlan)# shutdown

SW1(config)# interface range fa0/20 - 24
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 999
SW1(config-if-range)# shutdown
```

---

## 10. 한 줄 요약

> **VLAN은 물리 스위치 한 대를 논리적으로 여러 LAN으로 쪼갠다.** 같은 스위치라도 VLAN이 다르면 별개의 Broadcast Domain이며, VLAN 간 통신은 L3 라우팅이 있어야 가능하다. 다음 과제는 여러 스위치에 걸친 VLAN 전송 — 그 답이 Trunk(802.1Q)다.

---

## 관련 문서

- 충돌/브로드캐스트 도메인의 기초: [네트워크 스위칭 ① LAN과 스위치](/posts/network-05-lan-switch-domains/)
- (다음 편) Trunk와 Native VLAN, Voice VLAN: 802.1Q Trunk와 Native VLAN
- VTP의 위험성과 운영: VTP(VLAN Trunking Protocol)
- 다중 VLAN 간 라우팅: Multilayer Switch와 Gateway Redundancy
