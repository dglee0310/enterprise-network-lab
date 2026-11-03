## LAB 03 — STP Root Bridge 설계 및 VLAN별 경로 분산 (PVST+)

### 1. 설계 목적
- 단일 Root Bridge 구성 시 발생하는 백업 링크(DSW2 업링크)의 유휴 상태를 방지하고, VLAN별(VLAN 10, VLAN 20)로 Root Bridge를 상호 분산(Primary  Secondary)하여 L2 트래픽 로드 밸런싱(경로 분산)을 구현함.
- 향후 구축할 FHRP(HSRP Active) 장비와 STP Root Bridge 위치를 일치(정렬)시켜, 기본 게이트웨이로 향하는 트래픽이 불필요하게 DSW 간 Inter-link(Po1)를 우회하지 않도록 최적화된 L2 기반을 마련함.

### 2. 예상 결과
- Bridge Priority 조정을 통해 VLAN 10은 DSW1(Priority 4096), VLAN 20은 DSW2(Priority 4096)가 각각 Root Bridge로 선출된다.
- ASW1의 이중 업링크 활성화 시, ASW1의 포트가 VLAN별로 교차 차단(Forwarding  Blocking)되어 루프 없이 트래픽 경로가 분산된다.

 VLAN  STP Root Primary (4096)  STP Root Secondary (8192)  ASW1 Gi00 (DSW1 행)  ASW1 Gi01 (DSW2 행) 
 ---  ---  ---  ---  --- 
 VLAN 10 (USER)  DSW1  DSW2  Root  FWD  Alternate  BLK 
 VLAN 20 (INTERNAL)  DSW2  DSW1  Alternate  BLK  Root  FWD 

### 3. 실제 결과
- STP 수렴 및 포트 역할 확인 (`show spanning-tree`)
  - Extended System ID가 적용되어 VLAN 10은 Priority `4106`(4096+10), VLAN 20은 Priority `4116`(4096+20)으로 각 장비가 의도한 Root Bridge로 정상 선출됨을 확인.
  - ASW1에서 확인한 결과, 설계한 대로 VLAN 10은 Gi00(FWD)  Gi01(BLK), VLAN 20은 Gi01(FWD)  Gi00(BLK)로 교차 수렴되어 논리적 삼각형 루프가 차단됨.
  - PC1에서 각 게이트웨이 및 상위 IP로의 Ping 통신 정상 확인.
  - 캡처 이미지: `packet-analysis/stp/stp-root.png`
### 4. 결론
- PVST+의 Bridge Priority 튜닝을 통해 동일한 물리 토폴로지 내에서 VLAN 단위로 활성 포워딩 경로를 완벽히 분리·분산시켰다.
- 이를 통해 특정 링크의 대역폭 낭비를 없애고 가용성을 높였으며, 다음 단계인 HSRP Active-STP Root 정렬(Alignment)을 진행하기 위한 최적의 L2 경로 제어 환경을 성공적으로 검증 완료했다.