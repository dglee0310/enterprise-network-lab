## LAB 01-5 — HSRP (Hot Standby Router Protocol) 및 STP-FHRP Alignment

### 1. 설계 목적
- DSW1과 DSW2의 SVI(VLAN 10, 20)에 HSRPv1을 구성하여 기본 게이트웨이(Default Gateway)의 무중단 고가용성(HA)을 확보함.
- **STP Root와 HSRP Active 장비를 동일하게 일치(Alignment)**시켜, 호스트 트래픽이 DSW 간 LACP 링크(Po1)를 불필요하게 우회하지 않고 최단 L2/L3 경로로 라우팅되도록 최적화함.
- Priority(110) 및 Preempt(선점) 옵션을 활성화하여 장애 복구 시 지정된 Primary 장비가 즉각 Active 역할을 회수하도록 설계함.

### 2. 예상 결과
- **역할 분담 및 우선순위 (VLAN별 상호 이중화):**
  - **VLAN 10:** DSW1이 Active(Priority 110), DSW2가 Standby(Priority 100)
  - **VLAN 20:** DSW2가 Active(Priority 110), DSW1이 Standby(Priority 100)
- 각 PC(PC1: VLAN 10, PC2: VLAN 20)는 가상 IP(VIP)를 게이트웨이로 바라보며, 정상적으로 VIP Ping 및 Inter-VLAN 라우팅이 통신된다.
- Preempt 동작 시, 우선순위가 높은 라우터가 `Coup` 메시지를 송신하고 즉시 가상 MAC 주소로 `Gratuitous ARP(GARP)`를 브로드캐스트하여 하위 스위치의 CAM 테이블을 갱신한다.

| VLAN | Virtual IP (VIP) | HSRP Active (Pri: 110) | HSRP Standby (Pri: 100) | STP Root Bridge |
| :--- | :--- | :--- | :--- | :--- |
| **VLAN 10 (USER)** | 10.10.10.1 | **DSW1** | DSW2 | **DSW1 (정렬)** |
| **VLAN 20 (INTERNAL)**| 10.10.20.1 | **DSW2** | DSW1 | **DSW2 (정렬)** |

---

### 3. 실제 결과

#### 1) HSRP 상태 검증 (`show standby brief`)
- **DSW1:** VLAN 10(Group 10)은 `Active` 상태, VLAN 20(Group 20)은 `Standby` 상태로 정확히 선출됨.
- **DSW2:** VLAN 10은 `Standby`, VLAN 20은 `Active` 상태로 상호 백업 토폴로지가 정상 구성됨.
- 통신이 끊김 없이 유지될 줄 알았으나, Active로 전환시 텀이 발생해 핑이 도달하지 못하는 점을 관찰함.

#### 2) 호스트 통신 검증 (Ping)
- PC1(10.10.10.11) → 게이트웨이 VIP(10.10.10.1) 및 맞은편 VIP(10.10.20.1) Ping 성공.
- PC1(10.10.10.11) ↔ PC2(10.10.20.11) 간 Inter-VLAN 라우팅 통신 확인 (ICMP Reply 정상).

#### 3) Wireshark 패킷 심층 분석 (`packet-analysis/hsrp/`)
- **HSRP Coup 메시지 검증 (Frame 268):**
  - Preempt 활성화로 인해 우선순위가 높은 DSW1(Pri: 110)이 Active 자리를 인수할 때 `Op Code: Coup (1)` 프레임을 멀티캐스트(`224.0.0.2`, UDP 1985)로 전송함을 확인.
- **Gratuitous ARP (GARP) 동작 (Frame 271, 272):**
  - DSW1이 Active가 되는 즉시 가상 MAC 주소(`00:00:0c:07:ac:0a`, HSRP Group 10 표준 MAC)를 발신지로 하여 네트워크 전체에 브로드캐스트 GARP를 방출함.
  - 하위 Access 스위치들의 MAC Table 포트 매핑을 DSW1 방향으로 일괄 갱신시킴으로써 트래픽 누락을 방지함을 검증.
  - 캡처 이미지: `packet-analysis/hsrp/hsrp-failover-coup.png`

---

### 4. 결론
- HSRP를 통해 단일 장애점(SPOF)을 제거하고 게이트웨이 이중화를 성공적으로 구현했다.
- 앞서 LAB 03에서 구축한 **STP Root 경로와 HSRP Active 노드를 동일 장비로 완벽히 정렬(Alignment)**함으로써, L2 블로킹 포트로 인한 인터링크(Po1) 병목 현상 및 트래픽 역류 현상을 사전에 방지했다.