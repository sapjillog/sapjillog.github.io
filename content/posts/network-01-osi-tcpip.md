---
title: "네트워크 기초 ① 통신 모델 — OSI 7계층과 TCP/IP 4계층"
date: 2026-06-16
draft: false
categories: ["서버·인프라"]
tags: ["네트워크", "OSI", "TCP/IP", "캡슐화"]
summary: "OSI 7계층과 TCP/IP 4계층 모델을 현장 관점에서 정리한다. 서버/네트워크 지식 시리즈 1편."
---

> **개요** — 네트워크 통신은 "프로토콜"이라는 합의된 규칙 위에서 동작한다. 이 글은 프로토콜이 정의하는 세부 사항, OSI 7계층, TCP/IP 4계층, 그리고 각 계층의 헤더가 어떻게 캡슐화되는지를 정리한다.

---

## 1. 프로토콜이 정의하는 세부 사항

### 1.1 Encoding / Decoding

- 데이터를 전송 가능한 신호로 변환(Encoding) / 수신 신호를 데이터로 복원(Decoding)
- **NIC(Network Interface Card)** 가 수행

### 1.2 Encapsulation / Decapsulation

- **Encapsulation**: 규칙에 따라 데이터를 생성·전달하기 위해 헤더를 붙여 패키징하는 과정
- **Decapsulation**: 수신된 데이터를 확인·처리하기 위해 헤더를 벗기는 과정
- 송신 측에서는 상위 → 하위 계층으로 캡슐화, 수신 측에서는 하위 → 상위 계층으로 디캡슐화

### 1.3 Size (MTU)

- 네트워크는 긴 메시지를 그대로 보내지 않고 작은 조각(Frame/Packet)으로 쪼개 전달
- **MTU(Maximum Transmission Unit)**: 한 번에 전송 가능한 최대 크기 (Ethernet 기본 1500 bytes)
- 너무 길거나 너무 짧은 조각은 폐기됨
- 송신 호스트는 메시지를 최소·최대 크기에 맞게 분할(Fragmentation)
- 수신 호스트는 조각을 원래 메시지로 재조립(Reassembly)

### 1.4 Timing

| 항목 | 설명 |
| --- | --- |
| **흐름 제어 (Flow Control)** | 전송 속도 관리. 송수신 디바이스 간 정보 흐름을 조정해 수신자가 처리할 수 있는 속도로 보냄 |
| **응답 시간 초과 (Timeout)** | 허용된 시간 내 응답이 없으면 미수신으로 간주. 재전송 또는 무시 등의 후속 조치를 프로토콜이 규정 |
| **액세스 방법 (Access Method)** | 누가 언제 메시지를 보낼 수 있는지 결정. 동시 전송 시 **충돌(Collision)** 발생 → CSMA/CD 등으로 제어 |

### 1.5 Delivery Option

| 방식 | 통신 | 출발지 | 목적지 | 사용 환경 |
| --- | --- | --- | --- | --- |
| **Unicast** | 1:1 | me | you | LAN, WAN |
| **Broadcast** | 1:all | me | all | LAN only |
| **Multicast** | 1:N (특정 그룹) | me | group | LAN, WAN (주로 미디어 스트리밍) |

---

## 2. OSI 7 Layer

> 현장에서는 OSI 모델 그 자체로 통신을 처리하지 않는다. 복잡하고 대응이 느려 실제 인터넷 통신에서는 쓰이지 않으며, 눈에 보이지 않는 데이터 전달 방식을 **학습·설명하기 위한 용도**로 활용한다. 모든 인터넷 데이터 처리는 실질적으로 **TCP/IP 모델** 기반으로 동작한다.

| 계층 | 이름 | 역할 | 담당 영역 |
| --- | --- | --- | --- |
| L7 | Application | 응용 프로그램 인터페이스 | 개발자 영역 |
| L6 | Presentation | 인코딩·암호화·압축 | 개발자 영역 |
| L5 | Session | 세션 관리 | 개발자 영역 |
| L4 | Transport | 종단 간 신뢰성 있는 전송 | 네트워크 엔지니어 영역 |
| L3 | Network | 라우팅·논리 주소 | 네트워크 엔지니어 영역 |
| L2 | Data Link | 프레임 전송·물리 주소 | 네트워크 엔지니어 영역 |
| L1 | Physical | 전기 신호·매체 | 네트워크 엔지니어 영역 |

---

## 3. TCP/IP 4 Layer Model

> OSI의 상위 3개(L5/L6/L7)를 **응용 계층** 하나로 묶어 4계층으로 단순화. 데이터 전달용으로 1~4계층 사용. **TCP(L4)와 IP(L3)가 프로토콜 조합의 중심.**

| TCP/IP | OSI | 주요 프로토콜 | 주소 | 장비 |
| --- | --- | --- | --- | --- |
| **응용 계층 (Application)** | L5~L7 | HTTP, HTTPS, DNS, FTP, SSH, SMTP | Port (응용 식별) | - |
| **전송 계층 (Transport)** | L4 | **TCP**, UDP | Port Number | - |
| **인터넷 계층 (Internet)** | L3 | **IPv4**, IPv6, ICMP | IP Address | Router |
| **네트워크 액세스 계층** | L1~L2 | Ethernet(802.3), Wi-Fi(802.11) | MAC Address | Switch / Hub |

---

## 4. 계층별 상세 동작

### 4.1 응용 계층 (Application)

- 응용 프로그램이 동작하는 계층
- **Port 번호**를 가지고 동작 (응용 프로세스 식별자)
- Port 번호 범위
  - **Well-known Port (0 ~ 1023)**: HTTP(80), HTTPS(443), SSH(22), DNS(53) 등 표준 서비스
  - **Registered Port (1024 ~ 49151)**: IANA에 등록된 애플리케이션
  - **Dynamic / Ephemeral Port (49152 ~ 65535)**: 클라이언트 측 임시 포트

```
[ DATA ]
```

### 4.2 전송 계층 (L4)

- **주소**: Port Number (Source / Destination)
- **Source Port**: 클라이언트에서 무작위(Ephemeral 범위) 생성
- **Destination Port**: 목적지 컴퓨터의 응용 프로그램 포트 번호

```
[ L4 Header | DATA ]
   src port
   dest port
```

#### TCP vs UDP

| 항목 | TCP | UDP |
| --- | --- | --- |
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결 |
| 신뢰성 | 보장 (재전송·순서·체크) | 보장 안 함 |
| 흐름 제어 | O | X |
| 혼잡 제어 | O | X |
| 헤더 크기 | 20~60 bytes | 8 bytes |
| 속도 | 상대적으로 느림 | 빠름 |
| 용도 | HTTP, SSH, FTP, DB 등 | DNS, VoIP, 스트리밍, 게임 |

> **3-way handshake**: SYN → SYN+ACK → ACK 순서로 연결 수립
> **4-way handshake**: FIN → ACK → FIN → ACK 순서로 연결 종료

### 4.3 인터넷 계층 (L3)

- **주소**: IP Address (Source / Destination)
- **프로토콜**: IPv4(주력), IPv6
- **Source IP**: 자신의 IP (NIC 설정)
- **Destination IP**: 목적지 컴퓨터의 IP
- **라우터가 처리하며 종단 간 전송 중 IP는 변경되지 않음** (NAT 환경 제외)

```
[ L3 Header | L4 Header | DATA ]
   src IP
   dest IP
```

### 4.4 네트워크 액세스 계층 (L2 / 이더넷)

- **주소**: MAC Address (Source / Destination)
- **프로토콜**: Ethernet(802.3, 유선), Wireless(802.11, 무선)
- **MAC 주소 구조**: 48bit
  - 앞 24bit: **OUI (Organizationally Unique Identifier)** — 제조사 식별자 (IEEE 할당)
  - 뒤 24bit: **NIC-specific** — 제조사가 부여하는 고유 시리얼
- **Source MAC**: 자신의 MAC (NIC에 고정)
- **Destination MAC**
  - **LAN 통신**: 목적지 컴퓨터의 MAC (Switch가 처리)
  - **WAN 통신**: Hop by Hop으로 **Rewrite** (Router가 처리)
- 스위치는 L2 장비, 라우터는 L3 장비. L3 장비를 통과할 때 L2 헤더는 다시 만들어짐.

```
[ L2 Header | L3 Header | L4 Header | DATA ]
   src MAC
   dest MAC
```

### 4.5 물리 계층 (L1)

- 케이블·무선 매체를 통해 **전기적/광학적/전자기파 신호**로 비트 전송
- 매체: UTP, 광섬유, 무선(2.4/5GHz 등)
- 신호 인코딩 방식: NRZ, Manchester, 4B/5B 등

```
0101 1010 1100 0011 ... (비트 스트림)
```

---

## 5. 캡슐화 전체 흐름

송신 측:

```
[ DATA ]                                              ← Application
[ L4 Header | DATA ]                                  ← Transport
[ L3 Header | L4 Header | DATA ]                      ← Internet
[ L2 Header | L3 Header | L4 Header | DATA | FCS ]    ← Ethernet
0101011001110...                                      ← Physical
```

수신 측은 역순으로 디캡슐화하여 상위 계층으로 전달.

---

*서버/네트워크 지식 시리즈. 다음 편: IPv4 주소체계와 서브네팅.*
