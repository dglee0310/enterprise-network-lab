## LAB 01-3 — LACP (Link Aggregation Control Protocol) 및 Failover

### 1. 설계 목적
- DSW1과 DSW2 사이의 물리 링크 2개(Gi0/0, Gi0/1)를 논리적인 하나의 인터페이스(Po1)로 묶어 대역폭을 확장하고 링크 이중화를 구현함.
- 표준 프로토콜인 LACP(802.3ad) Active 모드를 통해 양단 간 동적 협상을 검증함.

### 2. 예상 결과
- LACPDU 패킷을 30초 주기로 교환하며 Port-Channel 1이 정상적으로 활성화(`SU`, 포트 플래그 `P`)된다.
- 802.1Q 트렁크가 적용되어 VLAN 10, 20 트래픽을 단일 논리 채널로 포워딩한다.

### 3. 실제 결과
- `show etherchannel summary` 확인 결과 `Po1(SU)` 및 두 포트 모두 `bundled(P)` 상태로 정상 결합됨.
- Wireshark 패킷 분석을 통해 Slow-Protocols 멀티캐스트(`01-80-C2-00-00-02`) 주소로 교환되는 LACPDU 프레임(Actor/Partner System ID 및 State)을 확인 완료.
- 캡처 이미지: `packet-analysis/lacp/lacp-aggregate.png`

### 4. 결론
- LACP 기반 포트채널 번들링이 정상 수렴되었으며, 다중 VLAN 트래픽을 수용할 수 있는 고가용성 L2 인터링크가 완성됨을 검증했다.
*(※ Failover 테스트 중 발생한 IOL 가상화 버그 및 상세 조치 과정은 별도 트러블슈팅 문서 `ts-01` 참조)