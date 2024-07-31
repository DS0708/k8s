# 여러 개의 서버를 이용해 쿠버네티스 클러스터를 설치하는 방법 - kubeadm

## Environment
- 쿠버네티스의 각종 기능을 제대로 사용하기 위해서는 최소 3대 이상의 서버를 준비해야 한다.
- 따라서 이번에는 1개의 마스터와 3개의 워커 노드로 구성된 테스트용 쿠버네티스 클러스터를 설치하는 방법을 배울 것이다.
- 먼저 각 서버에서 아래의 항목들이 준비 돼어야 한다.
    1. 모든 서버의 시간이 ntp를 통해 동기화 되어 있어야 한다.
    2. 모든 서버의 MAC 주소가 달라야 한다. 가상 머신을 복사해 사용할 경우, 같은 맥 주소를 가지는 서버가 존재할 수 있다.
    3. 모든 서버가 2GB 메모리, 2CPU 이상의 충분한 자원을 가지고 있는지 확인한다. 사용하는 설치 도구에 따라 요구하는 최소 자원의 크기가 조금씩 다를 수 있다.
    4. 모든 서버에서 메모리 Swap을 비활성화 해야한다. 메모리 스왑이 활성화 돼 있으면 컨테이너의 성능이 일관되지 않을 수 있기 때문에 대부분의 쿠버네티스 설치 도구는 메모리 스왑을 허용하지 않는다.
        - 스왑 비활성화 명령어
            ```
            swapoff -a
            ```

> 실제 서비스 운영 단계에 적용하려면 마스터 노드의 다중화와 같은 추가적인 설정이 필요할 수도 있지만, 간소화된 구성으로 쿠버네티스 클러스터를 설치해도 쿠버네티스의 핵심 기능을 사용하는 데에는 지장이 없어 크게 신경쓰지 않아도 된다.


## kubeadm으로 쿠버네티스 설치 (책을 참고하였지만 잘 안됨)
- 쿠버네티스는 일반적인 서버 클러스터 환경에서도 쿠버네티스를 쉽게 설치할 수 있는 kubeadm이라는 관리 도구를 제공한다.
- kubeadm은 쿠버네티스 커뮤니티에서 권장하는 설치 방법 중 하나이다.
- 또한 Minikube 및 kubespray와 같은 설치 도구도 내부적으로는 kubeadm을 사용하고 있기 때문에 현재도 활발히 개발되고 있는 쿠버네티스 설치 도구이다.
- kubeadm은 온프레미스 환경, 클라우드 인프라 환경에 상관 없이 일반적인 리눅스 서버라면 모두 사용 가능하다.

> AWS에서 Ubuntu기반 EC2 서버를 4개 생성한 다음에 진행할 것이다.

- Terraform으로 VPC 따로 구성하여 인스턴스 4개 배치하기
    ```
    provider "aws" {
    region = "ap-northeast-2"
    }

    resource "aws_vpc" "kube" {
    cidr_block = "172.31.0.0/16"
    tags = {
        Name = "kube-vpc"
    }
    }

    resource "aws_subnet" "public-b" {
    vpc_id            = aws_vpc.kube.id
    cidr_block        = "172.31.0.0/24"
    availability_zone = "ap-northeast-2b"
    map_public_ip_on_launch = true

    tags = {
        Name = "kube-subnet-public-b"
    }
    }

    resource "aws_internet_gateway" "kube" {
    vpc_id = aws_vpc.kube.id
    tags = {
        Name = "kube-igw"
    }
    }

    resource "aws_route_table" "kube" {
    vpc_id = aws_vpc.kube.id
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.kube.id
    }
    tags = {
        Name = "kube-rt"
    }
    }

    resource "aws_route_table_association" "kube" {
    subnet_id      = aws_subnet.public-b.id
    route_table_id = aws_route_table.kube.id
    }

    resource "aws_security_group" "kube" {
    vpc_id = aws_vpc.kube.id

    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }

    tags = {
        Name = "kube-sg"
    }
    }

    resource "aws_instance" "kube" {
    count         = 4
    ami           = "ami-062cf18d655c0b1e8"
    instance_type = "t3.medium"
    key_name      = "covyEc2"
    subnet_id     = aws_subnet.public-b.id
    vpc_security_group_ids = [aws_security_group.kube.id]

    tags = {
        Name = "kube-${count.index == 0 ? "master1" : "worker${count.index}"}"
    }

    user_data = <<-EOF
                #!/bin/bash
                apt update && apt install -y ntp
                systemctl enable ntp
                systemctl start ntp
                echo "vm.swappiness = 0" >> /etc/sysctl.conf
                sysctl -p
                swapoff -a
                EOF

    private_ip = count.index == 0 ? "172.31.0.100" : "172.31.0.10${count.index}"
    }

    output "instance_ips" {
    value = aws_instance.kube.*.public_ip
    }
    ```

- EC2 인스턴스 4개 생성하기
    - 인스턴스 이름과 private IP
    ```
    kube-master1  172.31.0.100
    kube-worker1  172.31.0.101
    kube-worker2  172.31.0.102
    kube-worker3  172.31.0.103
    ```

### 1. 쿠버네티스 저장소 추가
- 쿠버네티스를 설치할 모든 노드에서 다음 명령어를 차례대로 입력해 쿠버네티스 저장소를 추가한다.
    ```
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat << EOF > /etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    ```
- 이렇게 하려고 했으나 apt-key가 deprecated되어서 다른 방법으로 Kubernetes 저장소의 공개 키를 추가
    ```
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo tee /etc/apt/trusted.gpg.d/kubernetes.gpg >/dev/null
    ```
    - curl로 쿠버네티스 공개 키를 받아오기
    - sudo tee로 받아온 키를 /etc/apt/trusted.gpg.d/kubernetes.gpg 파일로 저장, tee 명령은 sudo와 함께 사용하여 루트 권한으로 파일을 작성할 수 있다.
    - '>/dev/null'은 tee 명령의 표준 출력을 억제하여 터미널에 출력되지 않도록 한다.
- 그 후 소스 리스트 파일 추가
    ```
    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    ```
    - /etc/apt/sources.list.d/kubernetes.list 내의 파일들은 APT가 패키지 데이터를 가져올 수 있는 저장소의 위치를 지정한다.
    - 'deb http://apt.kubernetes.io/ kubernetes-xenial main'은 위의 파일에 추가될 내용으로, 쿠버네티스 패키지가 포함된 저장소의 정보를 나타낸다.

### 2. containerd 및 kubeadm 설치
- 쿠버네티스 컨테이너 런타임 인터페이스(CRI)를 지원하는 구현체를 통해 컨테이너를 사용한다.
- containerd, cri-o(크라이-오) 등이 컨테이너 런타임 인터페이스를 통해 컨테이너를 제어할 수 있는 방법을 제공하며, 여기서는 containerd를 설치해 쿠버네티스와 연동하는 방법을 사용할 것이다.
- 도커를 설치하면 containerd가 함께 설치되므로 이번에는 도커에 포함된 containerd를 쿠버네티스와 연동해볼 것이다.

- 모든 노드에서 도커를 먼저 설치한다.
    ```
    wget -qO- get.docker.com | sh
    ```

> 근데 사실 여기서 설명하는 방법은 kubernetes가 컨테이너와 통신하기 위해서 컨테이너 런타임 인터페이스(CRI)가 존재해야 하는데
> CRI의 종류에는 containerd, cri-o 가 있다. 이 중에서 containerd는 도커에 자체적으로 내장되어 있기 때문에 도커를 설치하여
> 거기에 있는 containerd를 컨테이너 런타임 인터페이스로 사용하는 것이지만, 이슈가 한 두가지가 아니여서 그냥 공식문서를 참고하여
> containerd만 설치하기로 하였다.


## kubeadm 설치하기 (공식문서 참고)

### 시작하기 전에 확인 사항
- 호환되는 리눅스 머신. 쿠버네티스 프로젝트는 데비안 기반 배포판, 레드햇 기반 배포판, 그리고 패키지 매니저를 사용하지 않는 경우에 대한 일반적인 가이드를 제공한다.
- 2 GB 이상의 램을 장착한 머신. (이 보다 작으면 사용자의 앱을 위한 공간이 거의 남지 않음)
- 2 이상의 CPU.
- 클러스터의 모든 머신에 걸친 전체 네트워크 연결. (공용 또는 사설 네트워크면 괜찮음)
- 모든 노드에 대해 고유한 호스트 이름, MAC 주소 및 product_uuid.
- 컴퓨터의 특정 포트들 개방. 
- 스왑의 비활성화. kubelet이 제대로 작동하게 하려면 반드시 스왑을 사용하지 않도록 설정한다.

### MAC 주소 및 product_uuid가 모든 노드에 대해 고유한지 확인
- 사용자는 ip link 또는 ifconfig -a 명령을 사용하여 네트워크 인터페이스의 MAC 주소를 확인할 수 있다.
- product_uuid는 sudo cat /sys/class/dmi/id/product_uuid 명령을 사용하여 확인할 수 있다.

### 네트워크 어댑터 확인
네트워크 어댑터가 두 개 이상이고, 쿠버네티스 컴포넌트가 디폴트 라우트(default route)에서 도달할 수 없는 경우, 쿠버네티스 클러스터 주소가 적절한 어댑터를 통해 이동하도록 IP 경로를 추가하는 것이 좋다.

> 나는 위의 terraform 코드에 맞게 VPC를 하나 생성하고 거기에 subnet을 따로 할당하는 방식으로 생성하였다.

### 필수 포트 확인
```
nc 127.0.0.1 6443 -v
```

- terraform을 통해 ec2의 인스턴스 기존의 보안그룹에 6443 인바운드 규칙을 추가하였다.
    ```
    resource "aws_security_group_rule" "allow_kube_6443" {
    type              = "ingress"
    from_port         = 6443
    to_port           = 6443
    protocol          = "tcp"
    cidr_blocks       = ["0.0.0.0/0"]
    security_group_id = "sg-08f19b2bb1635c1cf"  # 기존 보안 그룹 ID
    }
    ```

### 컨테이너 런타임 설치
- 파드에서 컨테이너를 실행하기 위해, 쿠버네티스는 컨테이너 런타임을 사용한다.
- 기본적으로, 쿠버네티스는 컨테이너 런타임 인터페이스(CRI)를 사용하여 사용자가 선택한 컨테이너 런타임과 인터페이스한다.
- 런타임을 지정하지 않으면, kubeadm은 잘 알려진 엔드포인트를 스캐닝하여 설치된 컨테이너 런타임을 자동으로 감지하려고 한다.
- 컨테이너 런타임이 여러 개 감지되거나 하나도 감지되지 않은 경우, kubeadm은 에러를 반환하고 사용자가 어떤 것을 사용할지를 명시하도록 요청한다.

> containerd를 가장 많이 사용한다고 하여 설치하기로 했다.

### containerd 설치

> [containerd github](https://github.com/containerd/containerd/blob/main/docs/getting-started.md) 참고

1. containerd 설치
- [여기](https://github.com/containerd/containerd/releases)에서 해당 아키텍처에 맞는 버전을 선택하면 된다.
    ```
    curl -LO https://github.com/containerd/containerd/releases/download/v1.7.20/containerd-1.7.20-linux-amd64.tar.gz

    tar Cxzvf /usr/local containerd-1.7.20-linux-amd64.tar.gz
    ```
- [containerd.service](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service ) 설치하기
    ```
    wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

    mv containerd.service /etc/systemd/system/

    systemctl daemon-reload
    systemctl enable --now containerd
    ```
2. runc 설치
- [여기](https://github.com/opencontainers/runc/releases ) 참고하여 해당 아키텍처에 맞는 파일 선택, runc.<ARCH>
    ```
    curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.13/runc.amd64

    install -m 755 runc.amd64 /usr/local/sbin/runc
    ```
3. CNI plugins 설치
- cni-plugins-<OS>-<ARCH>-<VERSION>.tgz를 [여기](https://github.com/containernetworking/plugins/releases)를 참고하여 설치하면 된다.
    ```
    curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz
    
    mkdir -p /opt/cni/bin
    tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.1.tgz
    ```
4. containerd의 설정 파일 'config.toml'
- containerd의 설치 파일을 다운로드하여 설치했을 경우엔 해당 파일이 기본적으로 만들어지지 않기 때문에, 아래와 같이 containerd의 설정 파일을 만들어준다.
    ```
    mkdir -p /etc/containerd/

    containerd config default | sudo tee /etc/containerd/config.toml
    ```
- kubernetes를 위해 containerd를 사용할 것이라면 runC에서 cgroup을 사용하게 하기 위해 SystemCgroup 옵션을 true로 설정해야 한다.
    ```
    sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
    ```
    ```
    # cat /etc/containerd/config.toml | grep SystemdCgroup

    SystemdCgroup = true
    ```
- 설정 사항을 반영하기 위해 containerd를 재시작
    ```
    systemctl restart containerd
    ```
- 잘 실행되는지 확인하기
    ```
    systemctl status containerd
    ```

5. kubeadm, kubelet and kubectl 설치하기 [(참고)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
- 필요 패키지를 설치, Google의 공식 GPG 키 추가, Kubernetes APT 저장소 추가
    ```
    apt-get install -y apt-transport-https ca-certificates curl gpg

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
- APT 패키지 목록 업데이트 후 kubelet, kubeadm, kubectl 설치
    ```
    apt-get update
    apt-get install -y kubelet kubeadm kubectl

    apt-mark hold kubelet kubeadm kubectl

    systemctl enable --now kubelet
    ```
- kubeadm : K8s 클러스터를 쉽게 부트스트랩(초기 설정)할 수 있도록 도와주는 도구로, 이 커맨드를 통해 클러스터를 설정하고
관리하는 복잡한 과정을 단순화할 수 있다.
- kubelet : 모든 클러스터 노드에 설치되어 있으며, 쿠버네티스 클러스터의 필수 컴포넌트이다. 이는 각 노드에서 컨테이너의 실행과 관리를 담당한다.
- kubectl : 쿠버네티스 클러스터와 상호 작용하기 위한 커맨드 라인 도구이다. kubectl을 사용하여 클러스터 내부의 리소스를 조회, 생성, 수정 및 삭제할 수 있다.

> 아 그리고, 한국어로 된 공식 사이트를 참고하였더니 apt 패키지 목록이 업데이트 되지 않았다. 영어 공식 사이트는 최신화가 잘 되어 있는 것 같다. 가능하다면 영어 공식 사이트를 참고하자!!


### 쿠버네티스 클러스터 초기화
- 마스터 노드로 사용할 호스트에서 다음 명령어로 클러스터를 초기화 한다.
    ```
    kubeadm init --apiserver-advertise-address 172.31.0.100 --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock
    ```
    - `--apiserver-advertise-address` 옵션의 인자에 다른 노드가 마스터에게 접근할 수 있는 IP 주소를 환경에 맞게 입력한다. 위 예시는 Kube-master1 호스트에 접근할 수 있는 IP주소가 172.31.0.100인 경우다.
    - `--pod-network-cidr`은 쿠버네티스에서 사용할 컨테이너의 네트워크 대역이며, 각 서버의 네트워크 대역과 중복되지 않게 적절히 선택하면 된다.
    - 특정 버전의 쿠버네티스를 설치하려면 `--kubernetes-version 1.23.6`과 같이 kubeadm init 명령어에 버전 옵션을 추가하면 된다.
    - `--cri-socket unix:///var/run/containerd/containerd.sock` 옵션을 사용하면 쿠버네티스가 containerd의 컨테이너 런타임 인터페이스를 통해 컨테이너를 사용하도록 설정할 수 있다. 쿠버네티스는 도커와 containerd 양쪽이 모두 사용 가능할 경우 도커를 우선적으로 선택해 사용하므로 명시적으로 containerd를 사용하도록 설정해야 한다.

    > 추가로 쿠버네티스 클러스터가 제대로 작동하려면, 네트워크 트래픽을 포워딩할 수 있어야 한다. `/etc/sysctl.conf`파일을 `net.ipv4.ip_forward = 1` 이 부분이 주석 처리 되어 있는데 주석 처리를 지워주면 된다. 이 설정을 통해 IP 포워딩을 활성화 해야한다. 그 후 `sudo sysctl -p`로 시스템에 적용해줘야 한다.

- 초기화가 완료되면 다음과 같은 출력 결과를 확인할 수 있다.
    ```
    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 172.31.0.100:6443 --token wsmyme.0e34qjtl9jx2ivqe \
    --discovery-token-ca-cert-hash sha256:8584be3ec67e2aed8c0c8e91742e505ea60e20e55693d99277061173bd061798

    ```
- 이 중 다음과 같은 3줄의 명령어를 복사해 마스터 노드에서 실행한다.
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
- 맨 마지막에 출력된 명령어인 kubeadm join ~ 은 마스터를 제외한 워커 노드들에서 실행한다. 단, --cri-socket 옵션을 추가해줘야 한다.
    ```
    kubeadm join 172.31.0.100:6443 --token wsmyme.0e34qjtl9jx2ivqe --discovery-token-ca-cert-hash sha256:8584be3ec67e2aed8c0c8e91742e505ea60e20e55693d99277061173bd061798 --cri-socket unix:///var/run/containerd/containerd.sock
    ```
    > 참고로 위에서 설정한 네트워크 트래픽을 포워딩 설정도 각 워커 노드 별로 다 해줘야 한다. 위의 설정은 영구적으로 IP 포워딩을 활성화 하는 것이고 간단하게 하려면 다음 명령어 한 줄만 치면 된다.
    ```
    echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
    ```

### 컨테이너 네트워크 애드온 설치
- 쿠버네티스의 컨테이너 간 통신을 위해 flannel, weaveNet 등 여러 오버레이 네트워크를 사용할 수 있다.
- 이 중 calico를 설치해보자.
    ```
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    ```
- `--pod-network-cidr=192.168.0.0/16`이 아닌 별도의 IP대역을 입력했을 경우 calico.yaml을 직접 내려받아 CALICO_IPV4POOL_CIDR 환경변수의 각주를 해제한 후 적절한 IP 대역을 입력해야한다. 다음은 10.244.0.0/16인 경우다.
    ```
    wget https://docs.projectcalico.org/manifests/calico.yaml

    vi calico.yaml
    ```
    ```
    ...
                # chosen from this range. Changing this value after installation will have
                # no effect. This should fall within `--cluster-cidr`.
                - name: CALICO_IPV4POOL_CIDR
                value: "10.244.0.0/16"
                # Disable file logging so `kubectl logs` works.
                - name: CALICO_DISABLE_FILE_LOGGING
                value: "true"
    ...            
    ```
    ```
    kubectl apply -f calico.yaml
    ```
- 설치가 완료되었는지 확인하기 위해 다음 명령어를 입력한다. 전부 Running이라는 문구가 출력되었다면 정상적으로 설치가 완료된 것이다.
    ```
    # kubectl get pods --namespace kube-system
    NAME                                       READY   STATUS    RESTARTS   AGE
    calico-kube-controllers-5b9b456c66-s9ct6   1/1     Running   0          7m23s
    calico-node-49n6s                          0/1     Running   0          7m23s
    calico-node-ffjr2                          0/1     Running   0          7m23s
    calico-node-mt9dc                          0/1     Running   0          7m23s
    calico-node-p9cm9                          0/1     Running   0          7m23s
    ```
- `kubectl get nodes` 명령어를 사용하여 쿠버네티스에 등록된 모든 노드를 확인 가능하다.
    ```
    # kubectl get nodes
    NAME              STATUS   ROLES           AGE    VERSION
    ip-172-31-0-100   Ready    control-plane   129m   v1.30.3
    ip-172-31-0-101   Ready    <none>          90m    v1.30.3
    ip-172-31-0-102   Ready    <none>          88m    v1.30.3
    ip-172-31-0-103   Ready    <none>          87m    v1.30.3
    ```

### 삭제하기
- kubeadm으로 설치된 쿠버네티스는 각 노드에서 다음 명령어를 사용해 삭제할 수 있다.
    ```
    kubeadm reset
    ```

> 쿠버네티스 설치 도중 오류가 발생하거나 테스트용 쿠버네티스 클러스터를 삭제할 때 사용한다. 이전에
> 설치했던 쿠버네티스의 파일들이 /etc/kubernetes 디렉토리에 남아 있는 경우 kubeadm reset
> 명령어로 초기화한 뒤에도 설치에 실패할 수 있다. 따라서 kubeadm reset 후 설치 오류가 발생하면
> /etc/kubernetes 디렉토리를 다른 곳으로 옮기거나 삭제 후 다시 설치를 다시 시도 해야한다.