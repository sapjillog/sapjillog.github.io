---
title: "네트워크 스위칭 ⑦ EtherChannel — 링크 어그리게이션 (LACP·PAgP·Manual)"
date: 2026-06-27
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "스위칭", "EtherChannel", "LACP", "PAgP", "링크어그리게이션"]
summary: "여러 물리 링크를 하나의 논리 링크로 묶어 대역폭과 가용성을 확보하는 EtherChannel. LACP·PAgP·Manual 모드, 형성 매트릭스, 사전 조건, L3 EtherChannel, 부하 분산의 함정까지 정리한다. 서버/네트워크 지식 시리즈 11편."
---

> **개요** — EtherChannel(= Link Aggregation)은 **여러 물리 링크를 하나의 논리 링크로 묶어** 대역폭을 늘리고 가용성을 확보하는 기술이다. STP가 이중 링크 중 하나를 차단하던 문제를 우회할 수 있다.
>
> 사전 지식: [STP 기본 동작](/posts/network-09-stp/) · [RSTP와 STP 튜닝](/posts/network-10-rstp/)

---

## 1. 왜 EtherChannel인가

링크가 하나뿐이면 병목, 링크를 추가하면 STP가 하나를 BLK 처리해 대역폭은 그대로다. EtherChannel은 여러 링크를 묶어 **합산 대역폭**을 쓰면서 STP는 묶음 전체를 한 논리 링크로 인식한다.

| 효과 | 설명 |
| --- | --- |
| **Bandwidth 합산** | N개 링크 = N배 대역폭 (실제론 부하 분산 알고리즘에 따라 다름) |
| **Redundancy** | 한 가닥 끊겨도 나머지로 통신 지속 |
| **STP 안정성** | STP는 묶음 전체를 1개 링크로 봐서 BLK 없음 |
| **관리 단순화** | 단일 Port-Channel 인터페이스로 설정 일원화 |

---

## 2. EtherChannel 형성 프로토콜

| 프로토콜 | 표준 | 협상 | 권장 |
| --- | --- | --- | --- |
| **PAgP** | Cisco 전용 | 동적 | Cisco 전용 환경 |
| **LACP** | **IEEE 802.3ad / 802.1AX** | 동적 | **권장** (멀티 벤더) |
| **Manual (Static)** | - | 없음(강제 ON) | 협상 불가 시 |

---

## 3. PAgP 모드 (Desirable / Auto)

|  | Desirable | Auto |
| --- | --- | --- |
| **Desirable** | ✅ | ✅ |
| **Auto** | ✅ | ❌ |

> Auto-Auto는 둘 다 수동적이라 형성 안 됨.

```cisco
SW1(config)# interface range gi0/1 - 2
SW1(config-if-range)# channel-group 1 mode desirable
SW2(config-if-range)# channel-group 1 mode auto
```

---

## 4. LACP 모드 (권장, Active / Passive)

|  | Active | Passive |
| --- | --- | --- |
| **Active** | ✅ | ✅ |
| **Passive** | ✅ | ❌ |

```cisco
SW1(config)# interface range gi0/1 - 2
SW1(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# channel-group 1 mode passive
```

```cisco
SW1# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------
1      Po1(SU)       LACP        Gi0/1(P)   Gi0/2(P)
```

- `Po1(SU)` = Port-channel 1, **S**(Layer2)+**U**(in use)
- `Gi0/1(P)` = **P**(bundled in port-channel)

**LACP 추가 기능**: 최대 16개 링크 묶어 활성 8개+Standby(장애 시 자동 승격), `lacp system-priority`/`port-priority`로 Master·활성 링크 우선순위 조정.

---

## 5. Manual (Static) 모드

협상 없이 양쪽 모두 `mode on`으로 강제 형성.

```cisco
SW1(config-if-range)# channel-group 1 mode on
SW2(config-if-range)# channel-group 1 mode on
```

⚠️ keepalive가 없어 **단방향 케이블 장애를 감지 못함** → STP 루프 위험. 협상 가능한 환경이면 LACP 우선.

### 모드 호환 매트릭스 (전체)

| SW1 ↓ \ SW2 → | on | desirable | auto | active | passive |
| --- | --- | --- | --- | --- | --- |
| **on** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **desirable** | ❌ | ✅ | ✅ | ❌ | ❌ |
| **auto** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **active** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **passive** | ❌ | ❌ | ❌ | ✅ | ❌ |

> 서로 다른 프로토콜끼리는 절대 형성 안 됨(PAgP-LACP 혼용 불가).

---

## 6. EtherChannel 사전 조건

양쪽 인터페이스의 **Speed / Duplex / Switchport Mode / VLAN(Access면 동일 VLAN, Trunk면 동일 allowed·native VLAN) / MTU**가 모두 일치해야 형성된다.

설정 변경은 **반드시 Port-Channel 인터페이스에서** — 멤버 물리 인터페이스에 자동 반영된다.

```cisco
SW1(config)# interface port-channel 1
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
```

---

## 7. Layer-3 EtherChannel

묶은 논리 인터페이스에 IP를 부여해 라우티드 인터페이스로 동작(L3 백본·고대역폭 라우티드 링크).

```cisco
SW1(config)# interface range fa0/1 - 2
SW1(config-if-range)# no switchport               ! L3 모드로 전환
SW1(config-if-range)# channel-group 12 mode on
SW1(config)# interface port-channel 12
SW1(config-if)# ip address 192.168.12.1 255.255.255.0
```

> **⚠️ 순서 주의**: ① 멤버에 `no switchport` → ② `channel-group N mode on` → ③ Port-Channel에 IP 부여. `no switchport` 누락 시 L2 EtherChannel로 동작.

---

## 8. 부하 분산 알고리즘

묶인 링크 중 어디로 보낼지 **해시**로 결정한다(`src-mac`/`dst-mac`/`src-dst-mac`/`src-ip`/`dst-ip`/`src-dst-ip`/L4 포트 등).

```cisco
SW1(config)# port-channel load-balance src-dst-ip
```

> **⚠️ 흐름 단위 분산의 한계**: 같은 Src-Dst 쌍 트래픽은 **같은 링크로만** 흐른다. 즉 두 호스트 간 단일 대용량 전송은 하나의 링크 속도가 한계 — 여러 흐름이 있어야 합산 대역폭이 활용된다.

---

## 9. 점검 명령어

```cisco
SW1# show etherchannel summary               ! 가장 자주 사용
SW1# show etherchannel load-balance
SW1# show interfaces port-channel 1
SW1# show lacp neighbor                       ! LACP 이웃
SW1# show pagp neighbor                       ! PAgP 이웃
```

**비정상 플래그**: `(I)` stand-alone / `(s)` suspended(협상 실패) / `(H)` Hot-Standby / `(D)` down.

---

## 10. 자주 발생하는 실수

| 증상 | 원인 |
| --- | --- |
| 안 묶임 | 양쪽 mode 불일치(auto-auto, passive-passive) |
| 안 묶임 | 한쪽 LACP·한쪽 PAgP (프로토콜 혼용 불가) |
| 안 묶임 | Trunk allowed VLAN·Speed·Duplex 불일치 |
| L3 EC 동작 안 함 | `no switchport` 누락 |
| 멤버 추가 후 통신 끊김 | 멤버에 개별 설정 잔존 → Port-Channel 설정과 충돌 |

---

## 11. 전체 구성 예시 (Quick Reference)

```cisco
! ─── SW1 (LACP Active, Trunk EtherChannel) ───
SW1(config)# interface range gi0/1 - 4
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# switchport trunk allowed vlan 10,20,99
SW1(config-if-range)# switchport trunk native vlan 99
SW1(config-if-range)# channel-protocol lacp
SW1(config-if-range)# channel-group 1 mode active

SW1(config)# interface port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,99
SW1(config-if)# switchport trunk native vlan 99
SW1(config)# port-channel load-balance src-dst-ip

! ─── SW2 (LACP Passive) ───  ※ allowed/native VLAN 동일하게
SW2(config-if-range)# channel-group 1 mode passive
```

---

## 12. MLAG / vPC / VSS (참고)

단일 스위치가 SPOF인 점을 풀려고, EtherChannel을 **두 개의 다른 스위치**에 걸쳐 묶는 고급 기술. **vPC**(Cisco Nexus)·**VSS**(Catalyst 6500/4500)·**MLAG**(Arista 등)·**StackWise**(Catalyst 9300 등). 대형 데이터센터 디자인 필수.

---

## 관련 문서

- STP가 차단하던 이중 링크를 활성화: [STP 기본 동작](/posts/network-09-stp/)
- BPDU 보호 기능: [RSTP와 STP 튜닝](/posts/network-10-rstp/)
- (다음 편) IP 라우팅 기본 원리
