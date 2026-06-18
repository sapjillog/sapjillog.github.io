---
title: "네트워크 스위칭 ③ 802.1Q Trunk와 Native VLAN — 스위치 간 다중 VLAN 전송"
date: 2026-06-18
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "스위칭", "Trunk", "802.1Q", "Native VLAN", "VLAN Hopping"]
summary: "한 케이블로 여러 VLAN을 나르는 802.1Q Trunk의 원리, Native VLAN, Voice VLAN, DTP, VLAN Hopping 방어까지 정리한다. 서버/네트워크 지식 시리즈 7편."
---

> **개요** — 한 스위치 안에서 VLAN을 나누는 것까지는 [네트워크 스위칭 ② VLAN](/posts/network-06-vlan/)에서 다뤘다. 그런데 스위치가 여러 대인 환경에서 같은 VLAN끼리 어떻게 통신할까? 이 글은 **802.1Q Trunk**의 원리, Native VLAN, Voice VLAN, 그리고 보안 권고까지 정리한다.

---

## 1. 왜 Trunk가 필요한가

### 1.1 문제 시나리오

```
[ SW1: VLAN 10, 20, 30 ]──?──[ SW2: VLAN 10, 20, 30 ]
```

두 스위치를 한 가닥 케이블로 연결했을 때:
- VLAN 10 트래픽을 보내면 SW2가 *"이게 VLAN 10인지 20인지 어떻게 알지?"*
- 일반 Ethernet 프레임에는 **VLAN 정보가 없음**

### 1.2 두 가지 해결책

| 방법 | 동작 | 평가 |
| --- | --- | --- |
| VLAN별로 케이블 한 가닥씩 (Access 링크 N개) | VLAN 10용 케이블, 20용 케이블... | 비효율적, 확장성 X |
| **한 가닥에 VLAN 정보를 태그해서 전송 (Trunk)** | 프레임에 VLAN ID 첨가 | **표준 방식** |

→ **Trunk 링크**: 하나의 물리 링크로 **여러 VLAN 트래픽을 동시에 전달**

---

## 2. 802.1Q (dot1q) 표준

> IEEE 802.1Q — Ethernet 프레임에 **VLAN Tag(4 bytes)** 를 삽입하는 표준.

### 2.1 프레임 구조

```
일반 Ethernet:
[ Dst MAC | Src MAC | Type | Payload | FCS ]

802.1Q 태깅된 Ethernet:
[ Dst MAC | Src MAC | TAG (4 byte) | Type | Payload | FCS ]
                       ↑
                     VLAN ID 포함
```

### 2.2 TAG 필드 상세 (4 bytes)

| 필드 | 크기 | 설명 |
| --- | --- | --- |
| **TPID** (Tag Protocol ID) | 16 bit | 항상 `0x8100` (802.1Q 표시) |
| **PCP** (Priority Code Point) | 3 bit | QoS 우선순위 (0~7) |
| **DEI/CFI** | 1 bit | Drop 표시 |
| **VID** (VLAN ID) | **12 bit** | **VLAN 번호 (0~4095)** |

> 12 bit = 4096개 → VLAN 0과 4095는 예약 → **사용 가능 VLAN: 1 ~ 4094**

---

## 3. Trunk 동작 원리

```
[H1 VLAN 10] ─ Fa0/1 ─┐
                       SW1 ── 802.1Q Trunk ── SW2 ─ Fa0/2 ─ [H4 VLAN 10]
[H2 VLAN 20] ─ Fa0/2 ─┘                        └ Fa0/3 ─ [H5 VLAN 20]
```

**송신 측 SW1**:
1. H1(VLAN 10)에서 받은 프레임 → Trunk로 나갈 때 **VLAN 10 Tag 삽입**
2. Trunk 포트로 전송

**수신 측 SW2**:
1. Tag를 확인 → "이건 VLAN 10이군"
2. Tag 제거 후 해당 VLAN 포트로 전달

> 즉, Trunk는 VLAN의 **고속도로 차선**이고, 802.1Q Tag는 **차선 번호표**다.

---

## 4. Trunk 구성

### 4.1 기본 설정

```cisco
SW1(config)# interface fa0/14
SW1(config-if)# switchport trunk encapsulation dot1q    ! L3 스위치는 필요, L2 스위치는 dot1q 고정으로 생략 가능
SW1(config-if)# switchport mode trunk

SW2(config)# interface fa0/14
SW2(config-if)# switchport trunk encapsulation dot1q
SW2(config-if)# switchport mode trunk
```

> **참고 — `encapsulation dot1q`가 필요한 경우**
> 일부 L3 Catalyst(3560, 3750 등)는 ISL과 dot1q를 모두 지원하므로 명시적으로 dot1q를 지정해야 한다. 2960 같은 L2 전용은 dot1q만 지원하여 이 명령이 불필요할 수 있다.

### 4.2 Trunk 확인

```cisco
SW1# show interfaces fa0/14 switchport
Name: Fa0/14
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
```

```cisco
SW1# show interfaces trunk
Port    Mode    Encapsulation  Status   Native vlan
Fa0/14  on      802.1q         trunking 1

Port    Vlans allowed on trunk
Fa0/14  1-4094
```

### 4.3 허용 VLAN 제한

기본적으로 Trunk는 **모든 VLAN (1-4094)** 을 허용한다. 운영에서는 필요한 VLAN만 명시적으로 허용하는 것이 안전·효율적.

```cisco
! 특정 VLAN만 허용
SW1(config-if)# switchport trunk allowed vlan 10,20,50

! 추가
SW1(config-if)# switchport trunk allowed vlan add 60

! 제거
SW1(config-if)# switchport trunk allowed vlan remove 1

! 다시 전체 허용
SW1(config-if)# switchport trunk allowed vlan all
```

> **⚠️ 주의 — `switchport trunk allowed vlan`**
> `add`나 `remove` 없이 그냥 `switchport trunk allowed vlan X,Y`로 입력하면 **기존 허용 목록을 덮어쓴다.** 운영 환경에서는 항상 `add`/`remove`로 신중하게.

---

## 5. Native VLAN

### 5.1 정의

> **Native VLAN** = Trunk 링크에서 **태그 없이 전송되는** VLAN

```
일반 VLAN 트래픽: [ MAC | TAG (VLAN X) | Type | Payload ]
Native VLAN     : [ MAC | Type | Payload ]   ← Tag 없음
```

Cisco 장비의 기본 Native VLAN: **VLAN 1**.

### 5.2 왜 존재하나

- 초기 802.1Q 설계 시 **태그를 이해하지 못하는 구형 장비**와의 호환 목적
- Trunk 링크에 태그 없는 프레임이 들어오면 → Native VLAN으로 분류

### 5.3 양쪽 Native VLAN 일치 필수

Trunk 양 끝 스위치의 Native VLAN이 다르면 트래픽 누출 발생.

```cisco
SW1(config-if)# switchport trunk native vlan 99
SW2(config-if)# switchport trunk native vlan 99
```

> 양쪽 다 동일하게 설정하지 않으면 STP 등에서 `%CDP-4-NATIVE_VLAN_MISMATCH` 경고가 발생한다.

### 5.4 Native VLAN도 태그하기

보안상 Native VLAN까지 태그하는 것이 권장된다.

```cisco
SW1(config)# vlan dot1q tag native
SW2(config)# vlan dot1q tag native
```

---

## 6. Voice VLAN

### 6.1 시나리오

IP Phone은 보통 PC와 같은 자리에 둔다. IP Phone에는 2~3개의 포트가 있다.

```
[ Switch ] ── [ IP Phone ] ── [ PC ]
                  ↑                ↑
              VLAN 101 (Voice)  VLAN 100 (Data)
```

스위치 포트 1개로 PC와 IP Phone을 **동시에 다른 VLAN으로** 분리할 수 있다.

### 6.2 동작 원리

엄밀히 말하면 이 포트는 **하이브리드 Trunk** 동작:
- PC 트래픽: Native VLAN (태그 X) → Data VLAN
- IP Phone 트래픽: 802.1Q 태그 → Voice VLAN

### 6.3 구성

```cisco
SW1(config)# vlan 100
SW1(config-vlan)# name COMPUTER

SW1(config)# vlan 101
SW1(config-vlan)# name VOIP

SW1(config)# interface gi0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 100         ! PC용 데이터 VLAN
SW1(config-if)# switchport voice vlan 101          ! IP Phone용 Voice VLAN
```

---

## 7. DTP (Dynamic Trunking Protocol) — 주의

Cisco는 인접 스위치와 자동으로 Trunk를 협상하는 **DTP**를 제공한다.

### 7.1 모드별 동작

| 모드 | 의미 | 협상 |
| --- | --- | --- |
| `dynamic auto` | 상대가 trunk 요청 시만 trunk | 수동적 (기본값) |
| `dynamic desirable` | 상대에게 trunk를 적극 요청 | 능동적 |
| `trunk` | 강제 Trunk | 협상 X (한쪽이라도 강제면 trunk) |
| `access` | 강제 Access | 협상 X |

### 7.2 권장 — DTP 비활성화

운영에서는 **명시적 모드 + DTP Off** 권장.

```cisco
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport nonegotiate    ! DTP 비활성화
```

---

## 8. 보안 — VLAN Hopping 공격과 방어

### 8.1 VLAN Hopping이란

공격자가 Native VLAN의 약점 등을 이용해 **다른 VLAN의 트래픽을 가로채는** 공격.

- **Switch Spoofing**: 공격자가 자신을 스위치인 척 위장(DTP 협상 활용) → 자동 Trunk 형성 시 모든 VLAN 접근 가능
- **Double Tagging**: 두 개의 802.1Q 태그 삽입 → 첫 태그(Native VLAN)는 제거되고 두 번째 태그로 대상 VLAN 침투. Native VLAN 일치 + 태그 미적용 조건에서 성립.

### 8.2 방어 체크리스트

| # | 조치 |
| --- | --- |
| 1 | **DTP 비활성화** (`switchport nonegotiate`) |
| 2 | **사용자 포트는 `switchport mode access` 명시** |
| 3 | **Trunk의 허용 VLAN을 최소화** (`allowed vlan`) |
| 4 | **Native VLAN을 사용자 VLAN 1이 아닌 별도 번호로 변경** (예: 999) |
| 5 | **`vlan dot1q tag native`로 Native VLAN까지 태깅** |
| 6 | **미사용 포트는 별도 VLAN 격리 + `shutdown`** |
| 7 | **포트 단위 Storm Control** 활성화 |

---

## 9. 실무 빠른 점검

```cisco
SW1# show interfaces trunk                     ! Trunk 요약
SW1# show interfaces fa0/14 switchport         ! 상세 모드
SW1# show vlan id 10                           ! VLAN 10 소속 포트
SW1# show cdp neighbors                        ! 양쪽 장비 일치 확인
```

### 9.1 Trunk가 안 올라올 때 체크

| 증상 | 원인 |
| --- | --- |
| `Operational Mode: static access`로 남음 | `switchport mode trunk` 누락 |
| 한쪽만 trunk, 다른 쪽 access | 양쪽 모두 명시 필요 |
| 일부 VLAN만 통신 안 됨 | `allowed vlan`에 누락 |
| Native VLAN mismatch 로그 | 양쪽 Native VLAN 번호 불일치 |
| Trunk encapsulation 협상 실패 | L3 스위치에서 `encapsulation dot1q` 명시 누락 |

---

## 관련 문서

- VLAN 기본 개념: [네트워크 스위칭 ② VLAN](/posts/network-06-vlan/)
- (다음 편) VTP로 VLAN 동기화 및 위험성
- Trunk 위에서의 STP 동작
