## LAB 01-2 — VLAN / 802.1Q Trunk

### 1. 설계 목적
- VLAN별 브로드캐스트 도메인을 분리하고 DSW–ASW 사이 하나의 물리 링크에서 여러 VLAN을 전달함.

### 2. 예상 결과
- ASW1의 액세스 포트에서 수신한 VLAN 10 프레임이 DSW1로 전달되는 트렁크 구간에서 VLAN 10 태그를 포함한다.

### 3. 실제 결과
- PC1과 DSW1 SVI 간 ARP 및 ICMP 통신에 성공했으며 Wireshark에서 VLAN ID 10을 확인했다.
- 패킷 분석 파일: `packet-analysis/vlan/vlan10-trunk.pcapng`
- 캡처 이미지: `packet-analysis/vlan/vlan10-tag-analysis.png`

### 4. 결론
- VLAN 10의 L2 전달과 802.1Q 태깅이 의도대로 동작함을 검증했다.