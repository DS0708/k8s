# kops로 AWS에서 쿠버네티스 설치
- kops는 클라우드 플랫폼에서 쉽게 쿠버네티스를 설치할 수 있도록 도와주는 도구이다.
- kubeadm은 쿠버네티스를 설치할 서버 인프라를 직접 마련해야 하지만, kops는 서버 인스턴스와 네트워크 리소스 등을 클라우드에서 자동으로 생성해 쿠버네티스를 설치한다.
- 따라서 클라우드 플랫폼의 세부적인 리소스에 익숙하지 않아도 쉽게 인프라를 프로비저닝해 쿠버네테스 설치가 가능하다.

> kops는 AWS, GCP 등의 클라우드 플랫폼에서 설치를 지원하고 있다. 여기서는 래퍼런스가 많은 AWS에서 kops를 사용하는 방법을 설명할 것이다. 또한 Mac 환경에서 진행할 것이다.

## 1. kops 및 kubectl 실행 바이너리 내려받기
- kops와 kubectl을 다운 받아야한다. Mac환경에 Docker-Desktop가 설치되어 있어, 여기에서 쿠버네티스 활성화를 해줘서 kubectl은 사용 가능하여 kops만 설치 진행하였다.
    ```
    brew install kops
    ```
- 설치 확인
    ```
    kubectl version --client
    kops version
    ```

## 2. AWS 사용자 생성, 정책 연결 및 AWS CLI 설정
- AWS의 리소스를 프로비저닝하기 위한 aws CLI를 사용해야 한다.
- kops는 서버 인스턴스, 네트워크 리소스 등과 같은 인프라를 자동으로 생성하기 위해 aws CLI와 AWS 접근 정보를 사용한다.
- 이를 위해 aws CLI를 설치하고 적절한 역할이 부여된 user의 액세스 키를 aws CLI에 설정한다.
- aws configure 확인
    ```
    aws configure list
    ```

> 이 과정은 되어 있으므로 생략

## 3. S3 버킷에 쿠버네티스 클러스터의 설정 정보 저장
- kops는 쿠버네티스의 설정 정보를 s3 버킷에 저장하기 때문에 kops가 사용할 S3 버킷을 미리 생성해 둬야 한다. 다음 명령어로 s3버킷을 생성하고, 해당 s3 버킷의 versioning(비저닝)을 기록하도록 설정한다.
    ```
    aws s3api create-bucket --bucket covy-k8s-bucket \
    --create-bucket-configuration LocationConstraint=ap-northeast-2

    aws s3api put-bucket-versioning --bucket covy-k8s-bucket \
    --versioning-configuration Status=Enabled
    ```
- 쿠버네티스의 클러스터 이름과 S3 버킷 이름을 shell 환경 변수로서 설정
    ```
    export NAME=mycluster.k8s.local

    export KOPS_STATE_STORE=s3://covy-k8s-bucket
    ```
- 쿠버네티스를 설치할 EC2 인스턴스에 배포될 SSH 키 생성
    ```
    ssh-keygen -t rsa -N "" -f ./id_rsa
    ```
- 클러스터의 설정 파일 생성. 네트워크 플러그인은 다른 것을 사용해도 되지만 지금은 calico를 사용해 설치
    ```
    kops create cluster \
    --zones ap-northeast-2a \
    --networking calico \
    --ssh-public-key ./id_rsa.pub $NAME
    ```

## 4. 쿠버네티스 클러스터 옵션 변경
- 쿠버네티스를 생성하기 위한 각종 옵션을 수정해보자. 마스터와 워커 노드의 인스턴스 타입이나 워커 노드의 수를 조절할 수 있다.
- 먼저 다음과 같은 명령어로 현재 클러스터에 설정된 모든 인스턴스 그룹을 나열할 수 있다.
    ```
    $ kops get ig --name $NAME

    NAME				ROLE		MACHINETYPE	MIN	MAX	ZONES
    control-plane-ap-northeast-2a	ControlPlane	t3.medium	1	1	ap-northeast-2a
    nodes-ap-northeast-2a		Node		t3.medium	1	1	ap-northeast-2a
    ```
- 워커 노드의 옵션을 수정해보자. 노드의 개수를 수정할 수 있는 maxSize와 minSize를 수정해보자.
    ```
    kops edit ig --name $NAME nodes-ap-northeast-2a
    ```
    - maxSize와 minSize를 모두 3으로 설정하면 고정된 3개의 인스턴스만 생성된다.
- 마스터 노드의 설정은 다음과 같이 변경한다.
    ```
    kops edit ig --name $NAME control-plane-ap-northeast-2a
    ```

> 여기까지 진행하면 kops로 쿠버네티스를 생성하기 위한 준비가 끝난 것이다.

## 5. 쿠버네티스 클러스터 생성
- 다음 명령어를 입력하면 kops가 자동으로 서버 인스턴스, 네트워크 리소스 등을 생성해 쿠버네티스를 설치한다.
    ```
    kops update cluster --yes $NAME
    ```
- 쿠버네티스 클러스터를 준비하는 데 약 5~10분 정도가 소요된다.
- 충분한 시간이 흐른 후, 다음 명령어를 입력해 쿠버네티스에 접근할 수 있는 설정 파일을 가져온다.
    ```
    $ kops export kubeconfig --admin

    Using cluster from kubectl context: mycluster.k8s.local
    kOps has set your kubectl context to mycluster.k8s.local
    ```
- 다음 명령어로 노드의 목록과 쿠버네티스 버전을 출력해 설치가 정상적으로 완료됐는지 확인한다.
    ```
    kubectl get node
    ```
- kops로 생성한 쿠버네티스 클러스터 삭제
    ```
    kops delete cluster --name=$NAME --yes --state=$KOPS_STATE_STORE
    ```