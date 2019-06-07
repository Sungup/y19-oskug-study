# Overview of K8s (이관택 님 발표)

## Reference

- [An overview of kubernetes & (very) simple live demo](https://www.slideshare.net/GwanTaekLee/an-overview-of-kubernetes-very-simple-live-demo)

## Container Orchestration

### Container

Namespace & Cgroup에 의해 격리된 View

- cgroup: 시스템 자원들을 분리
- namespace: fs, process 등을 분리.

### CRI: Container Runtime Interface

k8s 에서 부르는 Container Runtime Interface. 다양한 Container를 제공하는
Registry 가 강점인 Docker를 가장 많이 사용함

- **장점**
  - 빠름, HA의 속도도 빠름.
  - 오픈소스 운영시 k8s가 알아서 관리해줌. (update 등등.)

### Ochestration

컨테이너 자동화해서 관리

- 자동배치 및 복제
- 그룹에 대한 로드 밸런싱
- 장애 복구
- 클러스터 외부에 서비스 노출
- 컨테이너 추가 또는 제거로 확장 및 축소
- 컨테이너 서비스 간의 인터페이스를 통한 연결및 네트워크 포트 노출 제어

Ochestration Service들

- **Docker Swarm**
  - Docker 내부에 내장된 Ochestration Tool.
  - 간단하지만 버그가 조금 있음. 개발용으로는 쓸만함.
- **Kuberenetes**
  - Google에서 개발, 2014년 9/9일 0.2 버전 Release.
  - github에서 인기 많음.

## Cluster Architecture

여러개 시스템을 붙여서 하나의 단위로 운영

### Master Node

Cluster 내 Container Runtime 관리를 전담하는 노드. Master가 죽더라도 관리만
전담하기 때문에 개별 Worker 노드들은 독립적으로 동작이 가능함. 단, Worker의
POD들이 문제가 발생하는 경우 제어/복구를 위해 Master를 3중화/5중화 이상으로
구성하여 운영함.

- **etcd**: Kubernetes DB, YAML 로 생성한 스크립트 내용을 저장하는 DB 서버.
  - HA 구성시 다중구성이 필요.
- **Scheduler**: 각 Node에 컨테이너 배치 관리.
- **Controll Manager**: Controller 생성 및 관리.
  - API Server를 통해 지속적인 모니터링해서 container runtime 제어.
- **API Server**: 사용자 & Node 간 통신.

### Worker Node

개별 Container Runtime들이 올라가서 동작하는 노드들.

- **kubelet**: Master와 통신, 노드 내의 컨테이너 관리
- **kube-proxy**: 트래픽 처리

## Installing Kubeadm

자세한 설치 방법은 [2. K8s Install](2_K8s_Install.md) 확인

1. 방화벽 포트 개방
   - k8s는 self에서만 사용하는 서비스용 포트 제외하고 모두 열기
   - pod network (calico) 관련된 port 대역 오픈.
     - overlay network 로 통신. pod 들간의 동신을 위해 사용.
     - 179 번 open, ip-in-ip protocol number 4
     - 성능면에서 괜찮다고 함.
2. CRI 설치: Docker 외 CRI-O, Containerd도 활용 가능
3. kubeadm, kubelet, kubectl 설치
4. SWAP off, Docker & Kubelet 실행
5. 클러스터 생성
   - kubeadm init --pod-network-cidr=***pod-network*** --apiserver-advertise-address=***master-ip***
   - pod 들이 구성할 네트워크의 대력 입력. B 클래스 대역으로 입력.
6. 3rd-party pod network 설치.
7. Join nodes.
   - kubeadm join ***master-ip***:***master-port***
     --token ***token*** --discovery-token-ca-cert-hash sha256:***hash***

8. Node Label 추가. (Optional)
   - kubectl label node ***hostname*** ***key***=***value***
   - pod 생성시 특정 node에 올리기 위한 용도.

## Pod

k8s에서 관리하는 가장 작은 배포단위로, Master에서 리소스를 갖고 있으며, 개별
Worker에는 실체가 없음. kube-proxy를 통해서 L/B가 관리됨.

- Volume: Container에서 사용하는 디스크
  - Host 상의 디스크, nfs, 램디스크 등 사용 가능
  - 다수의 container가 volume 을 공유할 수 있음.
- pause: K8s 전용 Container, container들을 하나로 묶어주는 역할.
  - Pod 내의 Container들은 동일 namespace를 공유하며, localhost로 접근 가능
  - Pod 간에는 Overlay Network로 연결됨.

## Controllers

Pod 들의 Resouce들을 관리. API Server를 모니터링하면서 manifest가 변경되는 것을
확인하면서 2개의 Controller를 생성. Pod 생성을 위한 다양한 Controller가 존재.

- **Repliction Controller/Replication Set**: Pod 복제용
- **Deployments**: RS의 상위 수준 리소스.
  - Rolling Update Strategy의 동작 방식때문에 RC/RS보다 상위 Resource가 필요.
    - Upate 시 Deployments 내의 신규 RS를 생성
    - Pod 단위로 이전 RS에서 하나씩 제거하며 신규 RS에 업데이트 버전을 생성
  - ip번호나 hostname들이 매번 바뀜
    - IP/Hostname을 바탕으로 한 통신은 이러한 이유로 의미없음
- **StatefulSets**: 특정 고정상태가 있는 pod
  - ip번호나 hostname을 주어 상태를 갖고 있는 pod 생성
  - Persistent Volume(PV)를 만들어 연결할 수 있음
    - Pod와 Volume 고정해서 연결하기 위한 용도
- **DaemonSet**: Node 당 1개씩만 생성하도록 보장. Replicas 옵션이 없음.
  - 시스템 로그/매트릭을 수집하는 용도로 사용하면 좋음.
  - Node Selector 기능도 있음
- **Jobs**: 1번 돌고 사라지는 용도.

## Service

- 논리적 Pos Set에 대한 접근 정책 정의. Pod 에 대한 **L4 L/B** 제공
- Proxy Mode는 3개 존재. 일반적으로는 Userspace 모드 활용.
- 다중 container 사용 시 Multi-Port 서비스 제공.
- DNS: 클러스터 내 접근하기 위한 Service name이나 FQDN을 활용.
  (IP가 휘발성이라서)
- 서비스 타입: Cluster IP를 많이 사용. Node Prot는 Public Cloud에서 활용.

## Ingress

- Context Level의 **L7 L/B**. Virtual Hosting, SSL/TLS 지원.
- 실제 Packet 처리를 위한 Ingress Controller (3rd Party App) 설치 필요.
- On-premiss 상황에서는 L4장비의 지원에 따라 구성하는 방법이 달라짐

### Nginx Ingress (Open Source)

- Load Balancer Type: Source IP가 유실됨. 테스트 용도로 활용.
- Ingress Controller Pod의 hostnetwork를 true로 해서 구성.
- 기존의 WAS 구조를 Web Server <-> App Server간의 처리를 ingress proxy
  레벨에서 처리가 가능하기 때문에 기존 대비 Web Server의 역할이 경량화

## Others

### Volumes

- emptyDIr
- hostPath
- configMap: 설정파일들을 DB에 저장해 두고 pod 올릴때 적절히 mount

### Secrets

- tls
- docker-registry: local only에서는 nexus 같은거 하나 만들어서 관리.

### RBAC: Roll based access controll

- Service account 생성
- role 또는 clusterrole을 만들어서 service account에 binding
- role: namespace 내, clusterrole: cross namespace 간
