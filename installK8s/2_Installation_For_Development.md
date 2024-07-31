# 개발 용도의 쿠버네티스 설치

## Docker Desktop 에서 쿠버네티스 사용
- 환경 설정에 Kubernetes에 들어가서 Enable Kubernetes의 체크박스를 체크한다.
- 활성화 한 뒤 restart하면 쿠버네티스와 관련된 도커 이미지들을 내려받기 때문에 시간이 조금 소요될 수 있다.
- 설치한 뒤에는 "kubectl version" 명령어를 입력하면 쿠버네티스 클라이언트와 서버의 버전이 출력된다.

> 단, Docker Desktop for Winodws/Mac에서는 일부 네트워크, 볼륨 기능이 제대로 동작하지 않을 수 있다. 따라서 
> Docker Desktop에서 제공하는 로컬 쿠버네티스는 테스트 용도로만 가볍게 사용하되, 앞으로 진행할 실습들은 완벽한 쿠버네티스
> 클러스터에서 진행하는 것을 권장한다.


## Minikube로 쿠버네티스 설치 (UTM에서 진행)
- Minikube는 로컬에서 가상 머신이나 도커 엔진을 통해 쿠버네티스를 사용할 수 있는 환경을 제공한다.
- Docker Desktop for Mac/Windows에 내장된 쿠버네티스와 마찬가지로 쿠버네티스의 기능을 간단히 사용해볼 수 있다는 장점은 있지만, 실제 운영환경에는 Minikube를 적용하기는 힘들뿐더러 쿠버네티스의 몇몇 기능도 사용할 수 없다는 단점이 있다.
- 따라서 가능하다면 여러 대의 서버로 쿠버네티스 클러스터를 구성하는 것이 좋다.

<br>

- Minikube는 가상 머신 또는 도커를 통해 쿠버네티스를 설치하기 때문에 Virtual Box 또는 도커 엔진이 미리 설치돼 있어야 한다.


### 기본 설정을 이용해 Virtual Box로 minikube 설치
1. Virtual Box 설치
    ```
    sudo apt-get install virtualbox
    ```
    > 참고로 arm은 안깔리는 것 같음
2. minikube, kubectl 내려받기
- minikube
    ```
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

    chmod +x minikube

    sudo mv minikube /usr/local/bin/
    ```
- kubectl
    ```
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

    chmod +x kubectl

    sudo mv kubectl /usr/local/bin/
    ```
    > 참고로 arm 아키텍처를 사용한다면 amd64 이 부분을 arm64로 바꿔서 진행하면 된다.
3. minikube 가상 머신 설치
- virtual box를 설치했다면 다음 명령어로 minikube 가상 머신을 생성 가능하다. minikube는 자동으로 minikube의 ISO 파일을 내려받아 설치한다.
    ```
    minikube start
    ```
- 특정 버전의 쿠버네티스를 설치하려면 다음과 같이 --kubernetes-version 옵션을 추가한다.
    ```
    minikube start --kubernetes-version v1.23.5
    ```
> 참고로 cpu 코어가 2개 이상일 때만 돌아간다.


### 리눅스 서버에서 가상 머신 없이 도커 엔진만으로 minikube 설치
- 도커 엔진을 미리 설치해 둔 리눅스에서는 가상 머신을 생성하지 않고도 Minikube를 이용해 쿠버네티스를 설치할 수 있다.
- 위의 '2. minikube, kubectl 내려받기'를 참고하여 minikube와 kubectl을 내려받은 뒤 실행
- conntrack, crictl 설치
    ```
    sudo apt install conntrack
    sudo apt install crictl
    ```
- k8s 1.24 버전 이후로는 k8s 에서 기본적으로 내부 연결 지원해주던 dockershim이 제거되어 cri-docker를 추가 설치하여 도커를 k8s에 연결하는 작업이 필요하다.
    ```bash
    # cri-docker 설치 후 /usr/local/bin에 옮기기
    curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd-0.3.14.arm64.tgz
    tar -xvzf cri-dockerd-0.3.14.arm64.tgz
    cd cri-dockerd/
    chmod +x cri-dockerd
    sudo mv ./cri-dockerd /usr/local/bin/

    # cri-docker 및 socket 활성화
    wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
    wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
    sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
    sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

    sudo systemctl daemon-reload
    sudo systemctl enable cri-docker.service
    sudo systemctl enable --now cri-docker.socket

    # cri-docker Active Check
    sudo systemctl restart docker && sudo systemctl restart cri-docker
    sudo systemctl status cri-docker.socket --no-pager

    # Docker cgroup Change Require to Systemd
    sudo mkdir /etc/docker
    cat <<EOF | sudo tee /etc/docker/daemon.json
    {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    },
    "storage-driver": "overlay2"
    }
    EOF

    sudo systemctl restart docker && sudo systemctl restart cri-docker
    sudo docker info | grep Cgroup

    # Kernel Forwarding 
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    br_netfilter
    EOF

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF

    sudo sysctl --system
    ```
- 그 후 실행하기
    ```
    minikube start --vm-driver=none
    ```
- 실행 완료 후 버전확인
    ```
    kubectl version
    ```
- 삭제하기
    ```
    minikube delete
    ```