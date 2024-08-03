# 구글 클라우드 플랫폼의 GKE로 쿠버네티스 사용하기
- kubeadm이나 kops로 쿠버네티스를 설치하는 것이 너무 어렵다면, 쿠버네티스의 설치부터 관리까지 전부 클라우드 서비스로서 제공하는 EKS, GKE 등의 매니지드 서비스를 사용할 수도 있다.
- 이번에는 클라우드 서비스로 제공하는 GKE(Google Kubernetes Engine)를 사용해볼 것이다.
- GKE는 클릭 몇 번으로 손쉽게 쿠버네티스를 설치하고 사용할 수 있게 해주며, 마스터 노드의 관리도 해준다.
- 그러나 GKE와 같은 매니지드 서비스는 편리하게 사용할 수 있지만 쿠버네티스의 세부 구조를 학습하기에는 적절하지 않다. 

> 대부분의 클라우드 서비스는 최초 사용에 한해 무료 평가판을 제공한다. 구글 컴퓨트 엔진도 12개월 동안 300달러를 사용할 수 있는 크레딧을 제공하고 있으므로 GKE의 결제가 부담스럽다면 무료 크레딧을 사용하는 것도 좋은 방법이다.

## 1. 가상 머신 클러스터 생성
- GKE 사이트에 접속해 구글 계정으로 로그인
- Kubernetes Engine 검색 후 들어가기
- 클러스터 만들기 클릭 -> Standard 버전으로 실행
- 기본 설정으로 일단 생성
    - 이름은 아무거나 
    - 서울 리전은 : asia-northeast3
- 5~10분 정도 걸림

## 2. 'gcloud' 설치 (Mac)
- 구글 클라우드 플랫폼의 명령어 준비
    ```
    brew install --cask google-cloud-sdk

    gcloud init
    ```
- gcloud init 후 url에 자동으로 들어가서 구글 아이디로 로그인
- GKE 클러스터 목록을 출력해 정상적으로 설치되었는지 확인하기
    ```
    gcloud container clusters list
    ```

## 3. 쿠버네티스 클러스터에 연결
- 해당 클러스터에 대한 kubeconfig 가져오기
    ```
    gcloud container clusters get-credentials covy-cluster-1 \
    --region asia-northeast3
    ```
- 현재 kubeconfig의 context 확인
    ```
    kubectl config current-context
    ```
- 안되어 있다면 GKE의 클러스터로 context 변경하기
    ```
    kubectl config use-context \
    gke_master-anagram-431402-a2_asia-northeast3_covy-cluster-1
    ```
- 클러스터가 생성되면 해당 클러스터에 들어가서 '연결'을 누르면 명령줄 액세스를 알려줌
    ```
    gcloud container clusters get-credentials covy-cluster-1 \
    --region asia-northeast3 \
    --project master-anagram-431402-a2
    ```
- 쿠버네티스의 노드의 목록을 출력해 클러스터를 정상적으로 사용할 수 있는지 확인
    ```
    kubectl get nodes
    ```














