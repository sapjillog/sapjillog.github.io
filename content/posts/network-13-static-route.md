---
title: "네트워크 라우팅 ② Static Route와 Floating Static — 정적 경로의 모든 것"
date: 2026-06-29
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "라우팅", "StaticRoute", "DefaultRoute", "FloatingStatic", "IPSLA"]
summary: "관리자가 직접 입력하는 가장 단순하면서 강력한 경로, Static Route. 기본 문법, Default Route, AD를 이용한 Floating Static 백업 경로, IP SLA+Object Tracking으로 만드는 진짜 fail-over, 그리고 양방향 누락·재귀 루프 같은 운영 함정까지 정리한다. 시리즈 13편(라우팅 2편)."
---

> **개요** — Static Route는 관리자가 라우팅 테이블에 직접 경로를 입력하는 가장 단순한 방법이다. 단순하지만 강력하며 Default Route·백업 경로·정책 라우팅의 기반이다. 일반 Static, Default Static, **Floating Static**, 그리고 운영의 함정을 정리한다.
>
> 사전 지식: [IP Routing 기본 원리](/posts/network-12-ip-routing/)

---

## 1. Static Route 기본

| 특성 | 값 |
| --- | --- |
| **AD** | **1** (Connected의 0 다음으로 높은 신뢰도) |
| **Metric** | 0 (비교 대상 아님) |
| 자동 학습 / 토폴로지 추적 | ❌ |

> **⚠️ Next-Hop 도달 가능성** — Static Route는 **Next-Hop으로의 경로가 보장**되어야 라우팅 테이블에 등록된다. Next-Hop이 다운되면 Static Route도 사라진다.

---

## 2. 명령 문법

```cisco
ip route <목적지 네트워크> <서브넷 마스크> <Next-Hop IP | 출력 인터페이스> [AD]
```

```cisco
! Next-Hop IP 방식 (권장)
R1(config)# ip route 192.168.2.0 255.255.255.0 192.168.12.2
! 출력 인터페이스 방식 (Point-to-Point에 적합)
R1(config)# ip route 192.168.2.0 255.255.255.0 Serial0/0
```

| 방식 | 특징 | 적합 환경 |
| --- | --- | --- |
| **Next-Hop IP** | 추가 ARP 조회 필요 | 다중 액세스(Ethernet) |
| **출력 인터페이스** | ARP 없이 직접 송신 | Point-to-Point(Serial 등) |

> Ethernet에 "출력 인터페이스 방식"을 쓰면 모든 패킷에 ARP 조회가 일어나 부하 증가 → 보통 Next-Hop IP 권장.

---

## 3. Default Route (Static Default)

라우팅 테이블에 일치 경로가 없을 때 **마지막으로 참조**하는 경로. `0.0.0.0/0` = "모든 서브넷".

```cisco
R1(config)# ip route 0.0.0.0 0.0.0.0 192.168.12.1
```

```cisco
R1# show ip route
Gateway of last resort is 192.168.12.1 to network 0.0.0.0
S*    0.0.0.0/0 [1/0] via 192.168.12.1
```

| 표시 | 의미 |
| --- | --- |
| `Gateway of last resort` | Default Route 설정됨 |
| `S*` | Static + Candidate Default (`*` = 마지막 안전망) |

**주요 사용처**: 사용자 라우터→ISP, 지사→본사, 라우팅 누락 시 폐기 대신 Default로 전송.

---

## 4. Floating Static Route (백업 경로)

같은 목적지에 Static이 둘이면 **AD가 낮은 쪽**이 등록된다. 백업 경로는 **AD를 수동으로 높여** 평소엔 비활성화한다.

```cisco
R2(config)# ip route 0.0.0.0 0.0.0.0 192.168.12.1          ! 기본 (AD=1)
R2(config)# ip route 0.0.0.0 0.0.0.0 192.168.23.3 100      ! 백업 (AD=100)
```

```cisco
R2# show ip route static
S*   0.0.0.0/0 [1/0] via 192.168.12.1        ← 평소 활성(AD 1)
! 주 경로 차단 후
S*   0.0.0.0/0 [100/0] via 192.168.23.3      ← 백업 활성화(AD 100)
```

> **⚠️ 함정 — 인터페이스 기반 Floating의 약점**: `ip route 0.0.0.0 0.0.0.0 Gi0/0 100`은 인터페이스가 down되지 않는 한 활성 유지 → Next-Hop이 죽어도 인터페이스가 up이면 남아 **fail-over가 동작 안 함**. Next-Hop IP 방식을 쓰거나 IP SLA로 보강.

---

## 5. IP SLA + Object Tracking — 진짜 fail-over

단순 Floating은 "Next-Hop은 살아있지만 그 너머 인터넷이 죽은" 상황을 감지 못한다. IP SLA로 **원격 상태(예: 8.8.8.8 도달성)**를 측정해 경로를 내린다.

```cisco
! ICMP 모니터링
R2(config)# ip sla 1
R2(config-ip-sla)# icmp-echo 8.8.8.8 source-interface gi0/0
R2(config-ip-sla-echo)# frequency 5
R2(config)# ip sla schedule 1 life forever start-time now

! Object Tracking
R2(config)# track 1 ip sla 1 reachability
R2(config-track)# delay down 10 up 10

! Static Route에 Tracking 연결
R2(config)# ip route 0.0.0.0 0.0.0.0 192.168.12.1 track 1   ! 기본
R2(config)# ip route 0.0.0.0 0.0.0.0 192.168.23.3 100       ! 백업
```

> `8.8.8.8`이 도달 불가하면 `track 1`이 down → 연결된 Static이 제거 → 백업 활성화. 검증: `show ip sla statistics 1`, `show track`.

---

## 6. 운영 패턴

```cisco
! 사용자 ↔ ISP
USER(config)# ip route 0.0.0.0 0.0.0.0 ISP-GATEWAY-IP

! 본사 ↔ 지사
HQ(config)# ip route 192.168.10.0 255.255.255.0 10.0.0.2    ! 지사1 LAN
BRANCH1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1          ! 본사로 모두

! 이중화 ISP (IP SLA+Tracking 권장)
HQ(config)# ip route 0.0.0.0 0.0.0.0 ISP1-GW                ! Primary(AD 1)
HQ(config)# ip route 0.0.0.0 0.0.0.0 ISP2-GW 100            ! Backup(AD 100)

! ECMP (동일 AD/Metric 두 개 → 부하 분산)
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.1
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.5
```

---

## 7. 주의사항과 함정

- **양방향 경로 누락**: 가는 경로만 있고 돌아오는 경로가 없으면 응답 패킷이 못 돌아와 ping 실패 → **항상 양방향** 설정.
- **Subnet Mask 실수**: `255.255.0.0`을 `/24` 의도로 잘못 넣으면 다른 경로까지 영향. `show ip route`로 prefix 길이 확인.
- **Recursive Routing Loop**: Next-Hop이 다시 첫 경로를 참조하면 무한 루프 → CEF가 `% ... cannot resolve` 경고.
- **출력 인터페이스 방식의 Proxy ARP 의존**: Ethernet에서 모든 목적지에 ARP → 다음 홉의 Proxy ARP 응답에 의존(성능·의존성 문제).

---

## 8. Static Route 장단점과 운영 권고

| 장점 | 단점 |
| --- | --- |
| 동작 단순·CPU 부하 거의 없음 | 토폴로지 변경 자동 추적 불가 |
| 예측 가능·보안(의도치 않은 학습 차단) | 대규모에서 관리 부담 폭증, 휴먼 에러 |

| 환경 | 권고 |
| --- | --- |
| 라우터 < 10대, 변화 적음 | Static만으로 운영 가능 |
| 대규모 / 동적 토폴로지 | **동적 라우팅 프로토콜** + 필요 시 Static 보강 |
| ISP 연결 | Default Static + (Floating + IP SLA) |
| 데이터센터 | OSPF/BGP + 특수 정적 경로(VRF·ECMP 등) |

---

## 9. 점검 명령어

```cisco
R1# show ip route static          ! Static만
R1# show ip route 0.0.0.0         ! Default 확인
R1# show ip route 192.168.2.2     ! 특정 IP 매칭 경로
R1# show ip sla statistics        ! IP SLA 상태
R1# show track                    ! Tracking 객체 상태
R1# debug ip routing              ! 라우팅 변경 디버그
```

---

## 관련 문서

- 라우터의 패킷 처리 흐름: [IP Routing 기본 원리](/posts/network-12-ip-routing/)
- (다음 편) 라우팅 프로토콜 분류와 경로 선택(AD·Metric·Longest Match)
- (이후) OSPF로 자동 학습
