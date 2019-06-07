# Kubernetes on CentOS7

CentOS7 에서 K8s 를 설치하는 방법 관련하여 Script 레벨로 정리하였습니다.

## References

- [CRI Installation](https://kubernetes.io/docs/setup/cri/)
- [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
- [Creating a single master cluster with kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

## Step 0. Prerequisit

우선 OS를 최신버전으로 올리고 SELinux의 기본 레벨을 낮춰줍니다. 또한 SWAP이
활성화 된 상태에서는 `kubeadm init`이 동작하지 않기 때문에 `/etc/fstab`에서
swap 파티션을 제거/주석처리 해 주시고 재부팅을 합니다.

```bash
# 최신 버전으로 centos7 및 tool들 설치
sudo yum update -y;
sudo yum install -y wget;

# SELinux를 enforcing에서 permissive로 낮춤.
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config;
sudo setenforce 0;
```

## Step 1. Install Docker

상기 CRI 항목상에서는 Docker, CRI-O, Containerd 등의 설치방법이 나와있으며,
이중에서 Docker 항목을 바탕으로 CRI 을 설치합니다.

설치 진행 시 `journalctl -xe`를 확인하면 **firewalld**에서 iptables 관련
warning 들이 발생하지만, 이후 설치 진행시 문제가 크게 발생 안하기 때문에
무시합니다.

```bash
# Install Pakages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2;
sudo yum-config-manager \
         --add-repo https://download.docker.com/linux/centos/docker-ce.repo;
sudo yum update && sudo yum install docker-ce;

# Make and change docker's daemon config json file.
sudo mkdir /etc/docker;
su root -c 'cat << EOF > daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF'

# Update service state
sudo mkdir -p /etc/systemd/system/docker.service.d;
sudo systemctl daemon-reload;
sudo systemctl restart docker;
sudo systemctl enable docker;
```

## Step 2. Update network settings

K8s의 본격적인 셋팅 이전에 network filter 관련 설정을 업데이트 합니다. 아래
진행시 최종 `sysctl --system` 적용 후, 아래 `net.bridge.bridge-nf-call-*`
항목들이 정상적으로 변경되었는지 확인해야 합니다.

추가로 일부 설치 시 hostname이 검색이 안되는 경우도 있을 수 있으니, hostname
맵핑을 `/etc/hosts`에 추가합니다.

```bash
# Update netfilter settings
su root -c 'cat << EOF > /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF'

sudo modprobe overlay;
sudo modprobe br_netfilter;
sudo sysctl --system;

# Add hostname in /etc/hosts. Please change the following XXX.XXX.XXX.XXX
# string to your ip address.
export API_SERVER_IP=XXX.XXX.XXX.XXX;

su root -c "echo ${API_SERVER_IP} $(hostname) >> /etc/hosts";
```

## Step 3. Installing kubeadm

아래와 같이 **Repository 설정**, **Firewall 설정**, **패키지 설치** 순서로
진행합니다.

```bash
# Add repository config
su root -c 'cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF'

# Open firewalld ports on the master node
sudo firewall-cmd --permanent --add-port=6443/tcp;
sudo firewall-cmd --permanent --add-port=2379-2380/tcp;
sudo firewall-cmd --permanent --add-port=10250/tcp;
sudo firewall-cmd --permanent --add-port=10251/tcp;
sudo firewall-cmd --permanent --add-port=10252/tcp;

# Open firewalld ports on the worker node
sudo firewall-cmd --permanent --add-port=10250/tcp;
sudo firewall-cmd --permanent --add-port=30000-32767/tcp;

# Install kubelet, kubeadm, and kubectl.
sudo yum install -y kubelet kubeadm kubectl --disableexclude=kubernetes;
sudo systemctl enable --now kubelet;

# Sync the kubelet's cgroup-driver with the docker settings.
su root -c 'echo KUBELET_EXTRA_ARGS=--cgroup-driver=systemd >> /etc/default/kubelet';

sudo systemctl daemon-reload;
sudo systemctl restart kubelet;
```

## Step 4. Initializing Kubernetes

`kubeadm init`으로 kubernetes 환경을 구축합니다. 아래 진행시 pod network는
`10.244.0.0/16`으로 셋팅해서 진행하겠습니다.

```bash
export API_SERVER_IP=XXX.XXX.XXX.XXX;
export POD_NET_CIDR=10.244.0.0;

# (Optional) Pull all kube images before setup.
sudo kubeadm config images pull;

# Run kubeadm init command with api-server-advertise-address and
# pod-network-cidr options.
sudo kubeadm init --pod-network-cidr=${POD_NET_CIDR}/16 \
                  --apiserver-advertise-address=${API_SERVER_IP};
```

`kubeadm init`을 실행한 직후 나오는 아래 내용중 `kubeadm join`의 내용을 따로
저장해 둡니다.

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

그리고 메세지 내용에 따라 아래 3줄을 그대로 다시 실행합니다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 5. Installing a pod network add-on

POD 네트워크 구축을 위한 Addon을 설치합니다. 본 문서에서는 `Calico` 애드온을
설치합니다. 단, Calico의 기본 설정파일은 pod-network를 `192.168.0.0/16`으로
설정되어 있기 때문에 바로 설치하지 않고 yaml파일을 다운받은 후 수정하여 다시
적용합니다.

```bash
# Open firewall prot for Calico
sudo firewall-cmd --permanent --add-port=179/tcp;

# Install Calico Network Add-on
export POD_NET_CIDR=10.244.0.0;

wget https://docs.projectcalico.org/v3.7/manifests/calico.yaml;
sed -i "s/192.168.0.0/${POD_NET_CIDR}/" calico.yaml;
kubectl apply -f calico.yaml;
```

## 번외편. Ingress Controller 설치

현재 관련내용 확인중입니다. ([Reference: Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/#prerequisite-generic-deployment-command))
정리가 되는 대로 내용 보강할 예정이며, 보강 전까지는 위의 공식문서를 참고해 주세요.

```bash
# Run Mandatory command
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml;

# Run Baremetal Specific Comamnd
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
```
