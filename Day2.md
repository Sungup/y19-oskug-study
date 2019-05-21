# Day 2

초안입니다. 추후 업데이트 다시 합니다.

## OS 재설치 방법

기본 JRE가 설치되어야 함.

1. Network 정보 복사 (아래 Appendix 참고)
2. Cafe24에 접속
   1. remote console -> console redirect 로 이동
   2. launch console 을 클릭해 파일을 받은 후 실행.
   3. java의 IP 할당에 대해 security 에서 예외설정 추가.
3. ipmi client 상태에서 hw 재부팅
   1. 재부팅 전에 cdrom 설정 (다운 먼저 받고)
   2. F11이나 Del 연타로 bios 로 빠져서 cdrom 으로 부팅
4. CentOS 설치
   1. 

추가 정리해서 올림.

실제 설치는 집에서 숙제.

## OpenStack Review

자료 공유...

LBAS 관련 부하가 커서 OCTAVIA가 새로 추가. Instance 를 통해 LBAS 관련 동작 처리.
BARBICAN: SSL 연동 등으로 사용.

HEAT & AODH: Autoscaling 가능하도록 하는 서비스. HEAT에서 임계치 설정. AODH에서 Alarm에 대해 Scaling 관련 처리 전달.

CEILOMETER: CPU/Memory/Netowrk/IO 등등 체크 (MongoDB -> 욥기)
RALLY: 셋팅이 정상적인지 테스트 (Pike 버전까지 나와있음...)

MAGNUM: Openstack에서 Container 처리

VM은 보통 KVM

Neutron: DHCP, Floating IP 관리.

Ctrl 관련된건 HA로 구성하지만 부하에 따라 분산 구성 가능
보통 DB의 부하가 큼. 분산구성해도 RabbitMQ를 쓰기 때문에 문제가 적으.

HA 구성시 Quorum 방식으로 사용 => 홀수로 늘어나는 쪽이 괜찮음.

Network 구성

- 순수 관리용망 1개
- API통신을 위한 망 1개
- Guest의 Out Traffic
- Compute 끼리 통신하기 위한 Guest 망

## Appendix. Network 관련 CentOS 설정 파일 위치

- **DNS**: /etc/resolv.conf
- **IP/Netmask/Gateway**: /etc/sysconfig/network-script/ifcfg-eth0
- **Hostname**: `hostname` 실행
