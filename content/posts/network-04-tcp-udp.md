---
title: "네트워크 기초 ④ TCP·UDP 상세 — L4 전송 계층의 모든 것"
date: 2026-06-16
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "TCP", "UDP", "전송계층", "혼잡제어"]
summary: "TCP 3-way/4-way, Seq/ACK, 흐름·혼잡 제어, TCP 상태, 포트, 트러블슈팅까지 L4를 한 번에 정리한다. 서버/네트워크 지식 시리즈 4편."
---

> **개요** — L4(Transport Layer)는 종단 간 데이터 전송을 담당한다. 이 글은 TCP의 3-way handshake·4-way termination, Sequence/ACK 동작, 흐름 제어, 혼잡 제어, 그리고 UDP와의 비교까지 한 번에 정리한다.
>
> 사전 지식: [네트워크 기초 ① 통신 모델](/posts/network-01-osi-tcpip/) (OSI/TCP-IP 모델, Port 번호 분류)

---

## 1. 전송 계층(L4)의 역할

| 역할 | 설명 |
| --- | --- |
| **종단 간 통신 식별** | Port 번호로 응용 프로세스 식별 |
| **신뢰성** | TCP만 보장. UDP는 보장 안 함 |
| **분할/재조립** | 큰 데이터를 Segment 단위로 |
| **흐름 제어** | 수신자의 처리 속도에 맞춰 송신 속도 조절 (TCP) |
| **혼잡 제어** | 네트워크 혼잡 시 송신 속도 축소 (TCP) |

L4의 두 프로토콜:

- **TCP** (Transmission Control Protocol) - 신뢰성 중시
- **UDP** (User Datagram Protocol) - 속도 중시

---

## 2. TCP vs UDP 비교

| 항목 | **TCP** | **UDP** |
| --- | --- | --- |
| 연결 방식 | Connection-Oriented (연결 지향) | **Connectionless** (무연결) |
| 신뢰성 | ✅ 보장 (재전송, 순서) | ❌ 보장 안 함 |
| **Sequencing** | ✅ 순서 복원 | ❌ |
| 흐름 제어 | ✅ Window Size | ❌ |
| 혼잡 제어 | ✅ Slow Start 등 | ❌ |
| 헤더 크기 | **20 ~ 60 bytes** | **8 bytes** |
| 속도 | 상대적으로 느림 | **빠름** |
| 통신 형태 | Unicast | Unicast / Multicast / Broadcast |
| 용도 | HTTP/S, SSH, FTP, SMTP, DB 등 | DNS, DHCP, VoIP, 스트리밍, 게임 |

### 2.1 비유

- **TCP**: 등기우편. 도착 확인, 재발송 가능, 비용 높음.
- **UDP**: 일반 엽서. 빠르고 가볍지만 잃어버려도 모름.

---

## 3. UDP 상세

### 3.1 헤더 구조 (8 bytes)

```
0               16              32 bit
┌───────────────┬───────────────┐
│  Src Port     │  Dst Port     │
├───────────────┼───────────────┤
│  Length       │  Checksum     │
└───────────────┴───────────────┘
│  Payload (데이터)              │
```

| 필드 | 크기 | 설명 |
| --- | --- | --- |
| **Src Port** | 16 bit | 송신 응용 프로그램 포트 |
| **Dst Port** | 16 bit | 수신 응용 프로그램 포트 |
| **Length** | 16 bit | UDP 헤더 + 데이터 길이 |
| **Checksum** | 16 bit | 오류 검출 (선택적, IPv4는 옵션) |

### 3.2 동작 특징

- **연결 설정 없음** → 즉시 데이터 전송
- **도착 여부 무관** → 응답 기대 안 함
- **재전송 불가** → 손실 시 응용에서 처리
- **순서 보장 안 됨** → 응용에서 직접 처리

### 3.3 UDP가 적합한 시나리오

| 응용 | 이유 |
| --- | --- |
| **DNS** | 짧은 요청-응답, 빠른 처리 |
| **DHCP** | Broadcast 기반 |
| **VoIP** | 손실보다 지연이 더 치명적 |
| **Video Streaming** | 약간의 손실 허용, 실시간성 우선 |
| **Online Gaming** | 낮은 지연 우선 |
| **SNMP** | 짧은 모니터링 메시지 |
| **NTP** | 시간 동기화 |

---

## 4. TCP 상세

### 4.1 헤더 구조 (20 ~ 60 bytes)

```
0       4       10      16              32 bit
┌───────────────┬───────────────┐
│  Src Port     │  Dst Port     │
├───────────────────────────────┤
│  Sequence Number (32 bit)     │
├───────────────────────────────┤
│  Acknowledgment Number (32 bit)│
├───┬───┬───────┬───────────────┤
│DOF│Rsv│ Flags │ Window Size   │
├───────────────┼───────────────┤
│  Checksum     │ Urgent Ptr    │
├───────────────────────────────┤
│  Options (가변, 0~40 bytes)    │
├───────────────────────────────┤
│  Payload (데이터)              │
└───────────────────────────────┘
```

| 필드 | 크기 | 설명 |
| --- | --- | --- |
| **Src Port / Dst Port** | 16 bit × 2 | 송수신 포트 |
| **Sequence Number** | 32 bit | 보낸 데이터 바이트 번호 |
| **ACK Number** | 32 bit | 다음 받기를 기대하는 Seq 번호 |
| **Data Offset** | 4 bit | 헤더 길이 (4 byte 단위) |
| **Flags** | 9 bit (6+ ECN/NS) | 제어 비트 (아래 참고) |
| **Window Size** | 16 bit | 수신 버퍼 여유 |
| **Checksum** | 16 bit | 오류 검출 |
| **Urgent Pointer** | 16 bit | URG flag 시 사용 |
| **Options** | 가변 | MSS, SACK, Window Scale 등 |

### 4.2 Flags 비트

| Flag | 의미 |
| --- | --- |
| **SYN** | 연결 시작 요청 (Synchronize) |
| **ACK** | 응답 확인 (Acknowledgment) |
| **FIN** | 연결 종료 요청 (Finish) |
| **RST** | 즉시 연결 종료 (Reset) |
| **PSH** | 즉시 응용 계층으로 전달 (Push) |
| **URG** | 긴급 데이터 (Urgent) |

---

## 5. TCP 3-Way Handshake (연결 설정)

> TCP는 데이터 전송 전 반드시 **연결 설정** 절차를 거친다.

### 5.1 흐름

```
[H1]                                    [H2]
  │                                       │
  │ ─── SYN (Seq=0) ─────────────────▶ │  ①
  │                                       │
  │ ◀──── SYN, ACK (Seq=0, Ack=1) ──── │  ②
  │                                       │
  │ ─── ACK (Seq=1, Ack=1) ────────▶ │  ③
  │                                       │
  │ ═══════ 연결 수립 완료 ═════════════ │
  │                                       │
  │ ─── 데이터 전송 시작 ──────────────▶ │
```

### 5.2 각 단계 의미

| Step | 송신자 | Flags | 의미 |
| --- | --- | --- | --- |
| ① | H1 | **SYN** | "당신과 이야기하고 싶습니다" |
| ② | H2 | **SYN + ACK** | "수락. 나도 대화하고 싶습니다" |
| ③ | H1 | **ACK** | "수락 확인" |

### 5.3 Sequence / ACK Number 동작

```
[H1 → H2]  SYN
  Seq = 0      (초기 시퀀스 번호 ISN)
  Ack = -

[H2 → H1]  SYN + ACK
  Seq = 0      (H2의 초기 시퀀스 번호)
  Ack = 1      ← H1의 Seq(0)에 대한 응답: "다음 받을 0+1=1"

[H1 → H2]  ACK
  Seq = 1      ← H1이 다음에 보낼 번호
  Ack = 1      ← H2의 Seq(0)에 대한 응답: "다음 받을 0+1=1"
```

> **SYN/FIN은 Sequence 1을 소비**
> 실제 데이터 0 byte를 보내도 SYN과 FIN은 각각 Seq +1로 카운트된다.

### 5.4 ISN (Initial Sequence Number)

- 처음 시작 시퀀스 번호는 **랜덤 값**으로 설정 (보안)
- 위 예시는 설명 편의를 위해 0/100 등을 사용

### 5.5 실제 Wireshark 시 보이는 모습

```
Frame 1: H1 → H2
  TCP Flags: 0x02 (SYN)
  Seq: 0 (raw=2871234567)

Frame 2: H2 → H1
  TCP Flags: 0x12 (SYN, ACK)
  Seq: 0 (raw=4192784512)
  Ack: 1 (raw=2871234568)

Frame 3: H1 → H2
  TCP Flags: 0x10 (ACK)
  Seq: 1
  Ack: 1
```

Wireshark는 사람이 보기 쉽게 ISN을 0으로 정규화해 보여준다.

---

## 6. TCP 4-Way Termination (보강 - PDF 누락)

> 연결을 끝낼 때는 **4단계**가 필요하다. 송수신 각각의 방향을 따로 닫기 때문.

### 6.1 흐름

```
[H1]                                    [H2]
  │                                       │
  │ ─── FIN (Seq=X) ──────────────▶   │  ①
  │                                       │
  │ ◀── ACK (Ack=X+1) ─────────────── │  ②
  │                                       │
  │      [H1→H2 종료, H2→H1 유지]      │
  │                                       │
  │ ◀── FIN (Seq=Y) ─────────────── │  ③
  │                                       │
  │ ─── ACK (Ack=Y+1) ─────────────▶  │  ④
  │                                       │
  │ ═══════════ 완전 종료 ═══════════════ │
```

### 6.2 각 단계 의미

| Step | 송신자 | Flags | 의미 |
| --- | --- | --- | --- |
| ① | H1 | **FIN** | "내 송신은 끝났다" |
| ② | H2 | **ACK** | "알겠다 (받음)" |
| ③ | H2 | **FIN** | "나도 송신 끝났다" |
| ④ | H1 | **ACK** | "알겠다 (확인)" |

### 6.3 왜 4번인가

- TCP는 **양방향 독립 연결** (Full-Duplex)
- H1→H2와 H2→H1의 송신 방향이 따로 종료 가능
- 2~3번 합쳐서 일어나기도 하지만 일반적으론 분리

---

## 7. TCP 연결 상태 (Connection State)

> 운영에서 `netstat`, `ss` 결과 해석에 필수.

```
                  ┌──────────────┐
                  │    CLOSED    │  ← 시작 상태
                  └──────┬───────┘
                Active   │   Passive
                Open     │   Open
                         ▼
       ┌─────────────────┴─────────────────┐
       │                                   │
   ┌───────┐                          ┌────────┐
   │SYN_SENT│                          │ LISTEN │  ← 서버 대기
   └───┬───┘                          └────┬───┘
       │ Send SYN+ACK                       │ Recv SYN
       │                                    │ Send SYN+ACK
       │                                    ▼
       │                              ┌─────────┐
       │                              │SYN_RECV │
       │                              └────┬────┘
       │                                   │ Recv ACK
       ▼                                   ▼
                ┌──────────────┐
                │ ESTABLISHED  │  ← 통신 중
                └───────┬──────┘
                        │
              Active    │   Passive
              Close     │   Close
                        ▼
    ┌───────────┐      │      ┌──────────────┐
    │FIN_WAIT_1 │      │      │  CLOSE_WAIT  │
    └─────┬─────┘      │      └──────┬───────┘
          │ ACK        │             │
          ▼            │             ▼
    ┌───────────┐      │      ┌──────────────┐
    │FIN_WAIT_2 │      │      │   LAST_ACK   │
    └─────┬─────┘      │      └──────┬───────┘
          │ FIN        │             │
          ▼            │             │
    ┌───────────┐      │             │
    │ TIME_WAIT │      │             │
    └─────┬─────┘      │             │
          │ 2MSL       │             │
          ▼            ▼             ▼
                  ┌──────────┐
                  │  CLOSED  │
                  └──────────┘
```

### 7.1 주요 상태

| 상태 | 의미 | 대표 시나리오 |
| --- | --- | --- |
| **LISTEN** | 서버가 연결 요청 대기 | `nginx`, `sshd` 등 |
| **SYN_SENT** | 클라이언트가 SYN 보내고 응답 대기 | 연결 시도 직후 |
| **SYN_RECV** | 서버가 SYN 받고 SYN+ACK 응답 후 ACK 대기 | 짧게만 머묾 |
| **ESTABLISHED** | 데이터 전송 가능 | 정상 통신 |
| **FIN_WAIT_1** | 능동적 종료 측이 FIN 보낸 직후 | 종료 시작 |
| **FIN_WAIT_2** | FIN_WAIT_1에서 ACK 받음. 상대 FIN 대기 | 정상 흐름 |
| **CLOSE_WAIT** | 수동적 종료 측이 FIN 받음. 응용 처리 중 | **여기 누적되면 비정상** |
| **LAST_ACK** | 수동 측이 자신의 FIN 보냄. ACK 대기 | 마지막 단계 |
| **TIME_WAIT** | 종료 후 일정 시간 대기 (2MSL) | 정상 종료의 흔적 |

### 7.2 TIME_WAIT의 의미

> **왜 TIME_WAIT가 필요한가**
> - 마지막 ACK가 손실되었을 때 상대의 FIN 재전송에 대비
> - 같은 4-tuple(Src IP/Port + Dst IP/Port)로 즉시 새 연결을 만들면 이전 연결의 잔여 패킷과 혼동 가능
> - 보통 **2 × MSL** (60초~4분, Linux는 60초)

대량의 TIME_WAIT가 누적되는 환경(HTTP 단방향 단명 연결 多)에서는:
- `tcp_tw_reuse` (Linux 커널 옵션)으로 재사용
- HTTP Keepalive로 연결 재사용
- Connection Pooling

### 7.3 CLOSE_WAIT 누적 = 응용 버그

> **⚠️ CLOSE_WAIT가 쌓이면**
> 응용 프로그램이 `close()`를 호출하지 않고 있다는 신호.
> 코드의 자원 해제 누락(파일 디스크립터 leak) 가능성 점검.

```bash
# Linux에서 상태별 연결 수 확인
$ ss -ant | awk '{print $1}' | sort | uniq -c
   1 LISTEN
   5 ESTAB
  234 TIME_WAIT
   2 CLOSE_WAIT       ← 비정상 누적이면 응용 점검
```

---

## 8. TCP 흐름 제어 (Flow Control) - 보강

> 송신자가 너무 빠르게 보내면 수신자 버퍼가 넘침. → **Window Size**로 조절.

### 8.1 Window Size

- 수신자가 자신의 가용 버퍼 크기를 매 ACK마다 알림
- 송신자는 ACK 받지 않고 **Window Size만큼만** 미리 보낼 수 있음

```
[H1 송신 버퍼]                          [H2 수신 버퍼]
  ┌────────┐                            ┌────────┐
  │ Data   │ ─── Seq=1, Win=8192 ──▶   │        │ (8KB 비어 있음)
  │ 1KB    │                            │        │
  │ Data   │ ─── Seq=2, Win=8192 ──▶   │ 1KB    │
  │ 2KB    │                            │ 사용   │
  │ ...    │                            │        │
  │        │ ◀── Ack=3, Win=4096 ────  │ 4KB    │ ← 버퍼 줄음
  │        │                            │ 사용   │
```

### 8.2 Zero Window

수신자 버퍼가 가득 차면 `Window = 0`을 광고 → 송신 일시 정지.
응용이 데이터를 읽어 버퍼가 비면 다시 광고.

### 8.3 Window Scale (Option)

- 16 bit Window 필드의 최대값 = 65,535 byte (64 KB)
- 고대역폭·고지연 환경(예: 100Gbps 대륙 간)에서는 부족
- **Window Scale Option**으로 최대 2^30 (1GB)까지 확장
- 3-way handshake 시 협상

---

## 9. TCP 혼잡 제어 (Congestion Control) - 보강

> 네트워크 자체가 혼잡할 때 송신 속도를 줄이는 메커니즘.

### 9.1 Congestion Window (cwnd)

송신자가 자체적으로 유지하는 송신 한도. 실제 송신량 = `min(rwnd, cwnd)`.

### 9.2 4단계 알고리즘

```
[ Slow Start ]
  cwnd = 1 MSS에서 시작
  ACK 받을 때마다 2배 (지수적 증가)
       ▼
[ ssthresh 도달 ]
       ▼
[ Congestion Avoidance ]
  cwnd를 1 MSS씩 (선형 증가)
       ▼
[ 패킷 손실 감지 ]
   ├ Duplicate ACK 3회 → Fast Retransmit + Fast Recovery
   └ Timeout → cwnd 1로, ssthresh 절반으로 (다시 Slow Start)
```

| 알고리즘 | 변종 | 특징 |
| --- | --- | --- |
| **Reno** | 기본 | Fast Retransmit/Recovery |
| **New Reno** | Reno 개선 | 연속 손실에 강함 |
| **Cubic** | Linux 기본 | 고대역폭에 최적 |
| **BBR** | Google 제안 | 대역폭·RTT 측정 기반, 손실 의존 X |

### 9.3 Linux 혼잡 제어 알고리즘 확인

```bash
$ sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = cubic

$ sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic bbr
```

---

## 10. 재전송 (Retransmission)

### 10.1 RTO (Retransmission Timeout)

- ACK 미도착 시 일정 시간 후 재전송
- RTO = RTT(왕복지연) 측정값에 기반해 동적 조정

### 10.2 Fast Retransmit

- **같은 ACK 번호가 3번 중복** 도착 → 손실 추정, 즉시 재전송
- RTO 만료를 기다리지 않음

### 10.3 SACK (Selective ACK, Option)

- 기존 ACK는 누적 방식 ("X까지 받았다")
- SACK는 **부분 수신 정보**도 전달 → 선택적 재전송
- 효율적인 손실 복구

---

## 11. MSS (Maximum Segment Size)

> TCP가 한 Segment에 담을 수 있는 **최대 데이터(헤더 제외) 크기**.

```
Ethernet MTU 1500
- IP 헤더 20
- TCP 헤더 20
= MSS 1460
```

### 11.1 MSS 협상

3-way handshake의 SYN 옵션에 자신의 MSS 광고. 양쪽 중 더 작은 값 채택.

### 11.2 Path MTU Discovery

경로 중 가장 작은 MTU에 맞춰 MSS 조정. DF(Don't Fragment) bit + ICMP "Fragmentation Needed" 메시지로 동작.

> **⚠️ 방화벽 ICMP 차단**
> ICMP를 모두 차단하면 PMTUD가 실패해 큰 패킷이 블랙홀에 빠질 수 있다. ICMP Type 3 Code 4 (Fragmentation Needed)는 허용해야 함.

---

## 12. Port 번호 관리

| 범위 | 분류 | 용도 |
| --- | --- | --- |
| **0 ~ 1023** | Well-known | 표준 서비스 (HTTP 80, HTTPS 443, SSH 22, DNS 53, FTP 21, SMTP 25) |
| **1024 ~ 49151** | Registered | 등록된 응용 (MySQL 3306, RDP 3389, PostgreSQL 5432) |
| **49152 ~ 65535** | Dynamic / Ephemeral | 클라이언트 임시 포트 |

### 12.1 자주 쓰는 포트

| 서비스 | 포트 | L4 |
| --- | --- | --- |
| FTP (Control) | 21 | TCP |
| SSH | 22 | TCP |
| Telnet | 23 | TCP |
| SMTP | 25 | TCP |
| DNS | 53 | UDP/TCP |
| DHCP | 67/68 | UDP |
| TFTP | 69 | UDP |
| HTTP | 80 | TCP |
| POP3 | 110 | TCP |
| NTP | 123 | UDP |
| IMAP | 143 | TCP |
| SNMP | 161/162 | UDP |
| LDAP | 389 | TCP |
| HTTPS | 443 | TCP |
| LDAPS | 636 | TCP |
| SMB | 445 | TCP |
| MySQL | 3306 | TCP |
| RDP | 3389 | TCP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |
| MongoDB | 27017 | TCP |

---

## 13. 실무 트러블슈팅

### 13.1 연결이 안 되는 케이스

| 증상 | 가능한 원인 |
| --- | --- |
| `Connection refused` | 대상 포트에 LISTEN 없음 |
| `Connection timeout` | 방화벽 차단, 라우팅 문제, 호스트 다운 |
| `Connection reset by peer` | 상대가 RST 전송 |
| `No route to host` | 라우팅 테이블 누락 |

### 13.2 상태 확인 명령

```bash
# Linux
$ ss -tulnp                              ! 리스닝 포트 + 프로세스
$ ss -ant state established              ! ESTABLISHED만
$ netstat -nat | grep TIME_WAIT | wc -l  ! TIME_WAIT 수

# Windows
> netstat -ano
> netstat -ano | findstr LISTENING

# 양쪽 모두
$ tcpdump -i eth0 -nn 'tcp port 443'
> Wireshark
```

### 13.3 패킷 캡처 해석

```
SYN 만 보이고 SYN+ACK 안 옴
  → 방화벽이 inbound 차단
  → 대상 호스트의 sshd/nginx가 실제로 LISTEN인가
  → SELinux/iptables 같은 호스트 방화벽

SYN, SYN+ACK는 정상, ACK 후 데이터 없음
  → 응용 응답 지연
  → 응용 버그
```

### 13.4 흔한 TCP 튜닝 (Linux)

```bash
# TIME_WAIT 재사용
$ sysctl -w net.ipv4.tcp_tw_reuse=1

# SYN backlog (대기 큐)
$ sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# Keepalive
$ sysctl -w net.ipv4.tcp_keepalive_time=600
$ sysctl -w net.ipv4.tcp_keepalive_intvl=60
$ sysctl -w net.ipv4.tcp_keepalive_probes=3

# 송수신 버퍼 크기
$ sysctl -w net.core.rmem_max=16777216
$ sysctl -w net.core.wmem_max=16777216
```

---

## 14. 보안 - TCP 관련 공격과 방어

| 공격 | 설명 | 방어 |
| --- | --- | --- |
| **SYN Flood** | 다수 SYN만 보내고 ACK 안 함 → 서버 자원 고갈 | SYN Cookies, SYN Proxy, Rate Limit |
| **TCP Reset Attack** | 위조된 RST 패킷으로 연결 강제 종료 | TCP Authentication Option (RFC 5925) |
| **Sequence Number Prediction** | 예측 가능한 ISN으로 세션 가로채기 | Random ISN (현재 표준) |
| **Port Scanning** | 열린 포트 식별 | Firewall, IDS/IPS, Port Knocking |

### 14.1 SYN Flood 방어 (Linux)

```bash
$ sysctl -w net.ipv4.tcp_syncookies=1
```

SYN Cookies는 SYN_RECV 상태를 메모리에 저장하지 않고 ACK Number에 정보를 인코딩 → 자원 고갈 회피.

---

## 15. 종합 비교 - 언제 무엇을 쓸까

```
신뢰성 필요? (손실 허용 X)
├─ Yes → TCP
└─ No
   │
   ├─ 실시간성이 더 중요? → UDP
   │  (VoIP, 게임, 라이브 스트리밍)
   │
   ├─ Broadcast/Multicast? → UDP
   │  (DHCP, 그룹 통신)
   │
   └─ 단순 짧은 요청-응답? → UDP
      (DNS, NTP, SNMP)
```

| 워크로드 | 권장 |
| --- | --- |
| 웹 (HTTP/S) | TCP |
| 파일 전송 | TCP |
| SSH | TCP |
| DB 연결 | TCP |
| DNS 일반 질의 | UDP |
| DNS 영역 전송 | TCP |
| VoIP | UDP (RTP) |
| 비디오 스트리밍 | UDP (RTP, RTSP) 또는 TCP (HLS, DASH) |
| HTTP/3 | UDP (QUIC) |

### 15.1 QUIC - 새 표준 (참고)

HTTP/3는 **QUIC** 위에서 동작. QUIC은 UDP를 기반으로 TCP의 신뢰성·암호화(TLS 1.3)·다중화를 모두 구현한 새 프로토콜.

```
HTTP/1.1 → TCP
HTTP/2   → TCP (다중화)
HTTP/3   → QUIC (UDP 기반)
```

이유: TCP는 OS 커널 의존도가 높아 변경이 느림. QUIC은 user space에서 빠르게 진화 가능.

---

## 16. 한 줄 요약

> **TCP는 신뢰성을 위해 손해 보고, UDP는 속도를 위해 신뢰성을 포기한다.**

각 응용의 요구사항을 이해하고 적절한 L4를 선택하는 것이 네트워크 설계의 출발점이다.

---

## 관련 문서

- L1~L4 헤더 전체 흐름: [네트워크 기초 ① 통신 모델](/posts/network-01-osi-tcpip/)
- IP 주소·NAT (L3의 짝): [네트워크 기초 ② IPv4 주소 체계와 서브네팅](/posts/network-02-ipv4-subnetting/)
- ARP·ICMP·Gateway (L2/L3 보조): [네트워크 기초 ③ ARP·ICMP·Gateway](/posts/network-03-arp-icmp-gateway/)
