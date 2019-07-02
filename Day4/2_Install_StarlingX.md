# Installing StarlingX AIO Simplex R2.0

## HW Requisite

- **Processor**: Minimum 8core, Xeon D-15XX family
- **Memory**: 64GB (But 32GB enable because of 32GB is the limitation for VM)
- **BIOS**
  - **Enable**: Hyper-Threading, Virtualization, VT for directed I/O
  - **Disable**: CPU C state control, Plug & play BMC detection
  - **Power & Performance Policy**: Performance
- **Storage**: 500GB

## SW Requirements

- **OS**: Ubuntu 16.04 LTS 64-bit (Documentation에 따르기 위해 Ubuntu 16.04로
  다시 설치)
- Git, KVM/VirtManager, Libvirt library, QEMU full-system emulation
- Tools project (maybe clone from git?), StarlingX ISO Image

## Preparing

우선 Documentation에 따르기 위래 Ubuntu의 16.04버전으로 OS를 다시 설치

```bash
# Disable firewall
sudo ufw disable;
sudo apt install -y git qemu-system qemu-kvm virt-manager libvirt-bin;
```

사용자 계정이 없는 경우 우선 stx 계정을 먼저 생성.

```bash
# On root user
useradd stx;
usermod -aG wheel stx; # for sudo command
passwd stx;
```

## Installing

최근버전의 starlingx iso 확보. 아래 링크에서 항상 최신버전으로 bootimage가 관리되기 때문에 아래 이미지 활용.

Reference: [All-in-one simplex R2.0](https://docs.starlingx.io/deploy_install_guides/latest/aio_simplex/index.html)
> Building the Software¶
> Follow the standard build process in the StarlingX Developer Guide.
>
> Alternatively a prebuilt iso can be used, all required packages are provided by the **StarlingX CENGN** mirror

```bash
wget http://mirror.starlingx.cengn.ca/mirror/starlingx/master/centos/latest_green_build/outputs/iso/bootimage.iso
```

이후 사용자 계정상에서 starlingx tools를 clone 한 다음 관련 dependency 파일들을
다운로드 및 설치. 이후 설치 진행을 위해 firewall을 비활성화 함.

```bash
# on stx user
git clone https://opendev.org/starlingx/tools;
cd tools/deployment/libvirt;
```

설치 진행 시 시스템의 용량이 부족하여 설정파일에서 메모리 용량을 변경해야 함.
기본 설정으로는 18GB가 할당되어 있으나 시스템 내 적당한 사이즈로 변경함.

```diff
diff --git a/deployment/libvirt/controller_allinone.xml b/deployment/libvirt/controller_allinone.xml
index 6f7272e..33b8c44 100644
--- a/deployment/libvirt/controller_allinone.xml
+++ b/deployment/libvirt/controller_allinone.xml
@@ -1,7 +1,7 @@
 <domain type='kvm' id='164'>
   <name>NAME</name>
-  <memory unit='GiB'>18</memory>
-  <currentMemory unit='GiB'>18</currentMemory>
+  <memory unit='GiB'>8</memory>
+  <currentMemory unit='GiB'>8</currentMemory>
   <vcpu placement='static'>6</vcpu>
   <resource>
     <partition>/machine</partition>
```

`controller_allinone.xml` 파일을 수정한 후 아래 명령어 Sequence를 통해 설치를
진행.

```bash
# Install dependencies
bash install_pakcages.sh;

# Setup OAM and management netowrks
bash setup_network.sh;

# Building XML for definition of virtual server
bash setup_configuration.sh -c simplex -i <starlingx iso image>;
```

`setup_configuration.sh` 을 실행 시 virt-manager가 실행됨. 이때 ssh 접속 시
`-X`으로 접속하게 되면 원격지의 virt-manager로 접근 가능함. 또한, virsh을
이용해 serial console로 접속이 가능하기에 원하는 대로 선택. 단, virsh로 접근할
경우 화면이 나오지 않는 경우 화살표를 위아래로 이동하면 선택 화면이 나타나게 됨.

```bash
sudo virsh console simplex-controller-0
```

1. **All-in-one Controller Configuration**
2. Install시 Connection Mode 선택
   - **Graphical Console**: virt-manager를 통한 설치시
   - **Serial Console**: virsh console을 통해 설치시
3. **STANDARD Security Boot Profile**

상기 3개의 선택지를 선택하고 나면 최대 약 20~25분간 설치가 진행됨.

이후, 최종 설치가 완료되면 로그인 화면에서 **sysadmin** 계정으로 로그인 함.
이때 사용하는 암호는 **sysadmin**이며, 로그인 후 바로 암호 변경 프로세스를
진행함. 내부적으로 기본 암호로 **St8rlingX\***으로 사용하기 때문에 본 내용에서도
동일한 암호를 사용함.

## Config controller

### Setup external connectivity

가상머신 상에서는 네트워크가 구성되어 있지 않기 때문에 내부 네트워크를 재구축.
이때 Host 상에서 `brctl show`와 `virsh domiflist simplex-controller-0`을 통해
Controller에 연결된 네트워크 `10.10.10.1` 상에 연결된 device의 mac을 확인한 후
controller 내부의 device를 확인함. 아래 진행에서는 **enp2s1**이 해당 network
device임.

```bash
sudo su
export CONTROLLER0_OAM_CIDR=10.10.10.3/24
export DEFAULT_OAM_GATEWAY=10.10.10.1
ifconfig enp2s1 $CONTROLLER0_OAM_CIDR
ip route add default via $DEFAULT_OAM_GATEWAY dev enp2s1
```

이후 `ansible`을 통해 playbook을 실행. 필요시 파일 내의 옵션값의 설정을 변경할
수 있음. ansible을 통해 진행 시 약 20~30 분간 설치가 진행됨.

```bash
ansible-playbook /usr/share/ansible/stx-ansible/playbooks/bootstrap/bootstrap.yml;
```

# Under Construction
