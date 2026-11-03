## 시스코 IOL 환경 LACP 버그로 인한 Split-Brain 및 통신 단절 장애 해결 

### 상황
 - DSW1-DSW2 간 Port-Channel(Po1, 2-Link LAG) 환경에서 단일 물리 링크(Gi0/0) 다운 시 잔여 링크로 Failover가 되지 않고 PC-스위치 간 통신이 전면 중단됨.
### 장애 증상 및 가설
 - 통신 불가: 잔여 링크(Gi0/1) 패킷 캡처 시 ARP/ICMP 패킷 유입 불가 (Blackholing).
 - 토폴로지 이상: 두 스위치 모두 자신이 Root Bridge라고 선언하는 Split-Brain 현상 발생.
### 원인 분석 및 해결
 -  원인 분석 (Cisco IOL 버그): Syslog에서 %EC-5-L3DONTBNDL2: Gi0/0 suspended 메시지 확인. 가상화 환경 특성상 포트가 즉시 Down되지 않고 Suspended(s) 상태에 머물면서 포워딩 엔진을 블록하고 BPDU 수신을 차단함.
 - 패킷 검증: Wireshark로 DSW2가 상위 루트 BPDU 단절 후 자신을 루트로 선언하며 Topology Change (TC) 플래그(0x01)를 송출하는 비정상 수렴 과정을 패킷 레벨에서 입증.
 - 해결 조치: 장애 포트에 명시적 shutdown 명령을 수동 적용하여 상태를 Down(D)으로 확정. 논리 인터페이스(Po1)가 정상 재정렬되며 STP 및 ICMP 통신 복구 완료.
 - 패킷 분석 파일: `packet-analysis/stp/stp-tc-packet.pcapng`
 - 캡처 이미지: `packet-analysis/stp/stp-tc-packet.png`

### 결과 및 배운 점
 - LACP 장애 조치 시 링크의 물리적 차단뿐만 아니라 내부 플래그 상태(show etherchannel summary) 검증이 최우선임을 인지함.