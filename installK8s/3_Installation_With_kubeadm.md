# 여러 개의 서버를 이용해 쿠버네티스 클러스터를 설치하는 방법 - kubeadm

## kubeadm 설치하기 (공식문서 참고)

### server IP Address
```
kube-master1  192.168.219.151
kube-worker1  192.168.219.152
kube-worker2  192.168.219.153
kube-worker3  192.168.219.154
```

> 가상머신을 이용해 4개의 ubuntu server를 생성

### 시작하기 전에 확인 사항
- 호환되는 리눅스 머신. 쿠버네티스 프로젝트는 데비안 기반 배포판, 레드햇 기반 배포판, 그리고 패키지 매니저를 사용하지 않는 경우에 대한 일반적인 가이드를 제공한다.
- 2 GB 이상의 램을 장착한 머신. (이 보다 작으면 사용자의 앱을 위한 공간이 거의 남지 않음)
- 2 이상의 CPU.
- 클러스터의 모든 머신에 걸친 전체 네트워크 연결. (공용 또는 사설 네트워크면 괜찮음)
- 모든 노드에 대해 고유한 호스트 이름, MAC 주소 및 product_uuid.
- 컴퓨터의 특정 포트들 개방. 
    ```
    sudo iptables -A INPUT -p tcp --dport 6443 -j ACCEPT
    sudo iptables -nL  # 확인
    ```
- 스왑의 비활성화. kubelet이 제대로 작동하게 하려면 반드시 스왑을 사용하지 않도록 설정한다.
    ```
    sudo swapoff -a

    # 스왑 영구 비활성화
    sudo sed -i '/swap/d' /etc/fstab
    
    # 확인
    grep swap /etc/fstab
    ```

### MAC 주소 및 product_uuid가 모든 노드에 대해 고유한지 확인
- 사용자는 ip link 또는 ifconfig -a 명령을 사용하여 네트워크 인터페이스의 MAC 주소를 확인할 수 있다.
- product_uuid는 sudo cat /sys/class/dmi/id/product_uuid 명령을 사용하여 확인할 수 있다.

### 네트워크 어댑터 확인
네트워크 어댑터가 두 개 이상이고, 쿠버네티스 컴포넌트가 디폴트 라우트(default route)에서 도달할 수 없는 경우, 쿠버네티스 클러스터 주소가 적절한 어댑터를 통해 이동하도록 IP 경로를 추가하는 것이 좋다.

> 나는 위의 terraform 코드에 맞게 VPC를 하나 생성하고 거기에 subnet을 따로 할당하는 방식으로 생성하였다.

### 필수 포트 확인
- 해당 포트가 사용중인지 확인한다. 
    ```
    $ nc 127.0.0.1 6443 -v
    nc: connect to localhost (127.0.0.1) port 6443 (tcp) failed: Connection refused
    ```

### 컨테이너 런타임 설치
- kubernetes가 컨테이너와 통신하기 위해서 컨테이너 런타임 인터페이스(CRI)가 존재해야 하는데, CRI의 종류에는 containerd, cri-o 가 있다. 
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
- containerd 인터페이스 사용할 수 있는 그룹 생성 및 할당
    ```
    sudo groupadd containerd
    sudo usermod -aG containerd covy
    sudo chown root:containerd /run/containerd/containerd.sock
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
    sudo systemctl restart containerd
    ```
- 잘 실행되는지 확인하기
    ```
    sudo systemctl status containerd
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
    sudo kubeadm init --apiserver-advertise-address 192.168.219.151 --pod-network-cidr=10.240.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock
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
    sudo kubeadm join 192.168.219.151:6443 --token b109o5.5ghoh9b2a92h3ou4 \
    --discovery-token-ca-cert-hash sha256:a8dae587b78c59809425ddbf962f8d5f7cf47ca1a4004c53a57c4683e3604250--cri-socket unix:///var/run/containerd/containerd.sock
    ```
    > 참고로 위에서 설정한 네트워크 트래픽을 포워딩 설정도 각 워커 노드 별로 다 해줘야 한다. 위의 설정은 영구적으로 IP 포워딩을 활성화 하는 것이고 간단하게 하려면 다음 명령어 한 줄만 치면 된다.
    ```
    echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
    ```

### 컨테이너 네트워크 애드온 설치
- Pod가 쿠버네티스 클러스터에 배포될 때, 각 Pod는 어떤 워커 노드에 배포될지 예측할 수 없으며,  각 Pod는 클러스터 내에서 고유한 IP 주소를 할당받는다.
- Pod 간 통신이 필요할 때, 어떤 Pod가 어느 노드에 있는지 상관없이 통신이 원활히 이루어지도록 관리할 수 있어야 한다.
- 이를 위해 네트워크 애드온의 설치는 필수적이며, 종류에는 Calico, Flannel, Weave 등이 있다.
- 이 중, 제일 많이 사용되는 Calico를 설치할 것이다.
    ```
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    ```
- `--pod-network-cidr=192.168.0.0/16`이 아닌 별도의 IP대역을 입력했을 경우 calico.yaml을 직접 내려받아 CALICO_IPV4POOL_CIDR 환경변수의 각주를 해제한 후 적절한 IP 대역을 입력해야한다. 다음은 10.244.0.0/16인 경우다.
    ```
    wget https://docs.projectcalico.org/manifests/calico.yaml

    sudo ivi calico.yaml
    ```
    ```
    ...
                # chosen from this range. Changing this value after installation will have
                # no effect. This should fall within `--cluster-cidr`.
                - name: CALICO_IPV4POOL_CIDR
                value: "10.240.0.0/16"
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