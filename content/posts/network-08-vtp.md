---
title: "네트워크 스위칭 ④ VTP — VLAN 자동 동기화와 'VTP 대참사'"
date: 2026-06-21
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "스위칭", "VTP", "VLAN", "Cisco", "Revision"]
summary: "한 스위치에서 만든 VLAN을 여러 스위치에 자동 동기화하는 VTP의 동작 원리와, Revision Number가 일으키는 'VTP 대참사', 그리고 실무 권고(Transparent 통일·VTPv3)까지 정리한다. 서버/네트워크 지식 시리즈 8편."
---

> **개요** — 스위치가 수십·수백 대인 환경에서 VLAN을 일일이 추가하는 것은 비효율적이다. VTP(VLAN Trunking Protocol)는 한 스위치에서 VLAN을 만들면 다른 스위치에 자동으로 동기화해주는 Cisco 전용 프로토콜이다. **편리한 만큼 사고도 잦다.** 이 글은 [네트워크 스위칭 ③ Trunk](/posts/network-07-trunk-native-vlan/)에 이어, VTP의 동작 원리와 운영에서 반드시 알아야 할 위험성을 정리한다.

---

## 1. VTP가 해결하는 문제

다수 스위치에 동일한 VLAN 정보를 유지해야 할 때:

```
[관리 전]
- SW1, SW2, SW3, SW4, SW5 각각에 vlan 10/20/30 수동 생성
- 새 VLAN 추가 시 5번 입력 → 누락 발생 가능
```

```
[VTP 도입 후]
- SW1(Server)에서 vlan 10 생성
- VTP가 SW2/3/4/5에 자동 전파 → 동기화
```

---

## 2. VTP 동작 모드

VTP는 3가지 모드로 동작한다.

| 항목 | **Server** | **Client** | **Transparent** |
| --- | --- | --- | --- |
| VLAN 생성·수정·삭제 | ✅ 가능 | ❌ 불가 | ✅ Local만 가능 |
| 받은 VTP 정보로 동기화 | ✅ | ✅ | ❌ |
| VTP Advertisement 전달 | ✅ | ✅ | ✅ (Pass-through) |
| VLAN.dat 저장 | ✅ | ✅ | ✅ (vlan.dat과 running-config 모두) |

> **참고 — Transparent 모드의 특수성**
> Local에서 VLAN을 만들고 변경할 수 있지만 **다른 스위치로 전파하지 않는다.**
> 받은 VTP 메시지는 그대로 다음 스위치에 통과시켜 준다(Pass-through).

---

## 3. VTP 동작 조건

| 조건 | 요구사항 |
| --- | --- |
| **VTP Domain** | 모든 스위치에 동일한 Domain Name 설정 |
| **Trunk 링크** | VTP Advertisement는 Trunk 위에서만 흐름 |
| **VTP Version** | 동일 Version (1/2/3) 사용 권장 |
| **Password** (선택) | 동일 비밀번호 설정 시만 동기화 |

---

## 4. VTP 구성

### 4.1 기본 상태 확인

```cisco
SW1# show vtp status
VTP Version capable             : 1 to 2
VTP version running             : 1
VTP Domain Name                 :
VTP Pruning Mode                : Disabled
VTP Operating Mode              : Server          ! 기본값
Configuration Revision          : 0
Number of existing VLANs        : 5
```

기본 모드는 **Server**, Domain은 **공란**.

### 4.2 Domain 설정

```cisco
SW1(config)# vtp domain SNET
Changing VTP domain name from NULL to SNET
```

Trunk로 연결된 스위치들이 이 Advertisement를 받으면 같은 Domain으로 동기화된다.

### 4.3 모드 변경

```cisco
SW2(config)# vtp mode client
Setting device to VTP CLIENT mode.

SW3(config)# vtp mode transparent
Setting device to VTP TRANSPARENT mode.
```

### 4.4 Password (보안)

```cisco
SW1(config)# vtp password Cisco123
```

> Password 미일치 시 VTP Advertisement는 무시된다.

### 4.5 Version 설정

```cisco
SW1(config)# vtp version 2
```

| Version | 특징 |
| --- | --- |
| **VTPv1** | 기본 |
| **VTPv2** | Transparent의 Token Ring 등 추가 지원, 기본 동작은 v1과 유사 |
| **VTPv3** | 인증·Primary Server 개념·MST 정보 전파 지원. 안전성 ↑ |

---

## 5. Revision Number — VTP의 핵심이자 함정

> VTP의 동기화는 **Configuration Revision**으로 결정된다.

### 5.1 동작 원리

- VLAN 정보가 바뀔 때마다 **Revision 번호 +1**
- 모든 스위치는 **자신보다 높은 Revision** 의 광고를 받으면 → 즉시 동기화

```
[Step 1] SW1 (Rev=5, vlan 10,20,30 보유)
[Step 2] SW2 (Rev=3, vlan 10,20 보유)
[Step 3] SW1이 Advertisement 전송 (Rev=5)
[Step 4] SW2: Rev=5 > Rev=3 → SW1의 정보로 덮어쓰기
[Step 5] SW2도 vlan 10,20,30 보유, Rev=5
```

### 5.2 함정 시나리오

```
[운영 환경]
- 본사 SW들: Domain=SNET, Rev=10, vlan 10/20/30 보유
- 새로 구입한 SW: Domain=SNET, Rev=99 (이전 테스트로 누적), vlan 1만 보유

→ 새 SW를 Trunk로 연결
→ 새 SW의 Rev=99가 본사 Rev=10보다 높음
→ 본사 모든 SW가 새 SW의 "vlan 1만 있음" 정보로 덮어쓰기
→ 전사 VLAN 정보 증발 = 대형 장애
```

> **⚠️ VTP 대참사**
> 실제로 이 사고는 수많은 기업에서 발생했다.
> 신규/중고 스위치를 Trunk로 연결하기 전에는 반드시 **Revision을 0으로 초기화**해야 한다.

### 5.3 Revision 초기화 방법

방법 1: **Domain Name 변경 후 원래대로**

```cisco
SW(config)# vtp domain TEMPDOMAIN     ! 다른 도메인으로 일시 변경
SW(config)# vtp domain SNET           ! 원래 도메인으로 복귀 → Rev=0
```

방법 2: **Transparent 모드로 전환 후 Server 복귀**

```cisco
SW(config)# vtp mode transparent
SW(config)# vtp mode server
```

방법 3: **VLAN.dat 삭제 + Reload**

```cisco
SW# delete flash:vlan.dat
SW# reload
```

---

## 6. VTP Pruning

> Trunk에 모든 VLAN 트래픽이 흐르면 불필요한 대역폭 낭비. VTP Pruning은 **수신 측 스위치에 해당 VLAN의 Active 포트가 없으면 Trunk에서 해당 VLAN 트래픽을 자동 제거**한다.

```cisco
SW1(config)# vtp pruning
```

| 효과 | 설명 |
| --- | --- |
| **장점** | 불필요한 Broadcast/Unknown Unicast 전파 차단 |
| **한계** | VTPv1/2는 동적 처리이므로 토폴로지 변경 시 일시적 미동작 가능. VTPv3는 미지원 |

---

## 7. 실무 권고 — VTP는 가급적 쓰지 마라

### 7.1 권고 사항

| # | 권고 |
| --- | --- |
| 1 | **운영 환경에서는 모든 스위치를 `vtp mode transparent`로 설정** |
| 2 | VLAN은 **각 스위치에서 직접 관리** (스크립트/Ansible 등 도구 활용) |
| 3 | 부득이 Server/Client 사용 시: **VTPv3 + Password + Primary Server** 필수 |
| 4 | 신규 스위치 투입 전 반드시 **Revision 0 초기화** |
| 5 | VTP 도메인 Password 설정 |

### 7.2 모든 스위치 Transparent로 통일하는 명령

```cisco
SW1(config)# vtp mode transparent
SW1(config)# vtp domain LOCAL              ! 일관성을 위해 도메인 자체는 설정
```

이 경우 VLAN 정보는 `vlan.dat`이 아닌 **running-config**에도 저장되어 가시성이 높아진다.

---

## 8. VTPv3 — 안전성 강화

VTPv3는 위 함정을 인식해서 만든 버전이다.

### 8.1 주요 개선점

| 항목 | VTPv1/2 | VTPv3 |
| --- | --- | --- |
| Primary Server | 없음 | **명시적 지정**. 변경은 Primary만 가능 |
| Authentication | 단순 Password | 강화된 Hidden Password |
| 전파 범위 | VLAN 1-1005만 | **1-4094 전체 + MST 정보까지** |
| Revision 사고 | 발생 | Primary 명시로 거의 차단 |

### 8.2 VTPv3 구성 예

```cisco
SW1(config)# vtp version 3
SW1(config)# vtp domain SNET
SW1(config)# vtp password Cisco@2026 hidden

SW1# vtp primary vlan       ! Primary Server 명시 (특수 명령, config가 아니라 exec 모드)
```

이렇게 하면 다른 스위치가 임의로 변경을 전파해도 동기화되지 않는다.

---

## 9. 점검 명령어

```cisco
SW1# show vtp status                  ! 모드/Domain/Rev/Version
SW1# show vtp counters                ! 송수신 통계
SW1# show vtp password
SW1# show vlan                        ! 현재 VLAN 목록
```

### 9.1 정상 동기화 확인

```cisco
SW1# show vtp status | include Revision
Configuration Revision          : 5

SW2# show vtp status | include Revision
Configuration Revision          : 5

SW3# show vtp status | include Revision
Configuration Revision          : 5
```

세 스위치의 Revision이 모두 일치하면 동기화된 상태.

---

## 10. 빈번한 오해와 정정

| 오해 | 정정 |
| --- | --- |
| "VTP는 VLAN 트래픽을 전송한다" | ❌ VTP는 **VLAN 정의 정보만** 동기화. 실제 VLAN 트래픽은 Trunk(802.1Q)가 운반 |
| "Transparent 모드는 VTP에서 빠진다" | ❌ Local VLAN 관리만 분리. **다른 스위치의 정보를 통과시키긴 함** |
| "Client는 안전하니까 마음 놓고 써도 됨" | ❌ Client도 더 높은 Rev를 받으면 동기화 → 동일 위험 |
| "VTP Domain이 같으면 자동 동기화" | ❌ **Trunk 링크** 가 있어야 함. Password도 일치해야 함 |
| "show running-config에서 VLAN이 보임" | ❌ Server/Client 모드에선 vlan.dat 별도 저장. Transparent만 running-config에 |

---

## 11. 사고 복구 절차

VTP 사고가 발생했다면:

```
1. Trunk 차단 (의심되는 신규 스위치와의 링크 shutdown)
   SW(config)# interface gi0/24
   SW(config-if)# shutdown

2. 백업된 VLAN 정보 복원
   - vlan.dat 백업이 있다면 TFTP로 복사
   - 또는 running-config 백업에서 vlan 명령 추출 후 수동 입력

3. 신규 스위치 Revision 초기화
   - delete flash:vlan.dat 후 reload
   - 또는 mode 변경 트릭

4. 재연결 전 show vtp status로 양쪽 Revision 확인
   → 신규 스위치 Rev=0 확인 후 Trunk 재개통

5. 사후 조치
   - VTPv3 + Primary Server 도입
   - 또는 모든 스위치 Transparent 모드로 전환
```

---

## 관련 문서

- VLAN 기본 개념: [네트워크 스위칭 ② VLAN](/posts/network-06-vlan/)
- Trunk(VTP의 전달 통로): [네트워크 스위칭 ③ Trunk와 Native VLAN](/posts/network-07-trunk-native-vlan/)
- (다음 편) STP — 스위칭 루프 방지의 기본 원리
