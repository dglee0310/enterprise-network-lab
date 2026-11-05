## LAB 05 — OSPF 동적 라우팅 및 L2/L3 장애 우회

### 1. 설계 목적
- DSW1, DSW2와 R-HQ 사이에 OSPF Area 0을 구성하여 내부 서브넷(VLAN 10, 20)을 동적으로 라우팅함.
- R-HQ ↔ DSW 링크는 Point-to-Point로 설정하여 DR/BDR 선출을 생략하고 수렴 속도를 높임.
- 사용자 SVI에는 `passive-interface`를 적용하여 불필요한 OSPF Hello 송출을 막음.
- **설계 선택:** DSW 간에 L3 링크를 추가로 만들지 않고, 기존 L2 트렁크(Po1)를 그대로 유지함. 장애 시 **하행은 OSPF 재수렴**, **상행은 HSRP Interface Tracking**으로 처리하도록 역할을 나눔.

#### 참고: DSW 간 OSPF 직접 연결 vs HSRP Tracking
- **DSW 간 OSPF 연결 시:** DSW1의 업링크가 끊겨도 DSW2를 거쳐 우회할 수 있으므로 HSRP Active를 바꿀 필요가 없음.
- **HSRP Tracking을 선택한 이유:**
  1. Po1은 이미 여러 VLAN을 실어 나르는 L2 트렁크임.
  2. L3 링크나 Transit VLAN을 추가로 만들지 않아도 됨.
  3. L2(HSRP/STP)와 L3(OSPF)의 역할을 분리하는 일반적인 캠퍼스 설계 방식이며, 업링크 장애 시 HSRP Active를 넘기는 것이 동작을 이해하기 쉬움.

### 2. 예상 결과
- **ECMP:** DSW1, DSW2가 VLAN 10/20을 광고할 때 R-HQ 기준으로 두 경로의 Cost(2)와 AD(110)가 같으므로, 하행 트래픽이 양쪽으로 분산된다.
- **LSR은 필요한 쪽만 보낸다:** OSPF는 DBD 교환 시 LSA 시퀀스 번호를 비교하여, 자신에게 없거나 상대가 더 최신인 LSA만 LSR로 요청한다. 따라서 R-HQ가 이미 최신 정보를 갖고 있다면 DSW2에게 LSR을 보내지 않을 것이다.

### 3. 실제 결과

#### 1) OSPF 인접 관계 형성
- Hello로 네이버를 발견한 뒤, P2P 링크이므로 DR/BDR 선출 없이 ExStart로 진입.
- DBD로 서로의 LSDB 목록을 교환한 후 `FULL` 상태 도달.
- **LSR 확인 (Frame 29, 30):**
  - DSW2는 DSW1(`1.1.1.2`)의 Router-LSA를 받기 위해 LSR(29번)을 보냈고, R-HQ가 LSU(30번)로 응답함.
  - R-HQ는 DSW2에게 LSR을 보내지 않음 → DSW2의 DBD를 확인한 결과 추가로 필요한 LSA가 없었기 때문.
  - 캡처 이미지: `packet-analysis/ospf/ospf-adjacency-lsa.png` 

#### 2) 정상 상태: ECMP 확인
- R-HQ에서 `show ip route 10.10.10.0` 결과:
  ```text
  O  10.10.10.0/24 [110/2] via 10.255.0.6, GigabitEthernet0/1
                   [110/2] via 10.255.0.2, GigabitEthernet0/0
  ```
- DSW1 방향(`Gi0/0`)과 DSW2 방향(`Gi0/1`) 두 경로가 모두 라우팅 테이블에 올라가 로드밸런싱이 동작함을 확인.

#### 3) 업링크 장애 시 우회 확인
- **장애 유발:** DSW1의 업링크(`Gi0/0`)를 끊음.
- **하행 (외부 → PC):**
  - R-HQ에 `%OSPF-5-ADJCHG: Neighbor Down (Dead timer expired)` 로그 발생.
  - `via 10.255.0.2` 경로가 삭제되고 `10.255.0.6`(DSW2) 단일 경로로 전환됨.
- **상행 (PC → 외부):**
  - DSW1의 `track 1 interface line-protocol`이 업링크 다운을 감지하여 HSRP Priority를 110 → 90으로 낮춤.
  - DSW2가 Preempt로 HSRP Active가 되어 상행 트래픽을 자신의 업링크로 전달함 → DSW1에서 트래픽이 갇히는 블랙홀 방지.

### 4. 결론
- R-HQ ↔ DSW 간 OSPF P2P 구성으로 DR/BDR 없이 인접 관계를 맺고, 내부 서브넷에 대해 ECMP가 동작함을 확인했다.
- 패킷 캡처를 통해 LSR이 필요한 LSA에 대해서만 요청된다는 것을 확인했다.
- DSW 간 L3 링크 없이도 **상행은 HSRP Tracking, 하행은 OSPF 재수렴**으로 업링크 장애를 우회할 수 있음을 검증했다.
- 단, Standby → Active 전환 시 플래핑이 발생하는 단점이 있다.
*(※ Failover 테스트 중 발생한 패킷 유실 상황은 별도 트러블슈팅 문서 `ts-02` 참조)*