# kops로 GCP에서 쿠버네티스 설치

> kops, gcloud, kubectl이 미리 설치되었다고 가정

## gcloud login
- `gcloud init` 혹은 `gcloud auth application-default login`로 로그인
- 프로젝트 설정, 프로젝트 ID는 GCP 홈페이지에서 확인가능
```
gcloud config set project <Project ID>
```

## ssh 연결을 위한 rsa key 생성 
```
ssh-keygen -t rsa -N "" -f ./id_rsa
```

> 여기서 만든 RSA Key로 ssh 연결을 하기 위해서는 GCP에 별도로 등록이 필요함

## 클러스터 설정을 state store에 저장
- Cloud Storage에 버킷 만들기 (서울 region에 생성)
```
gsutil mb -l asia-northeast3 gs://covy-kubernetes-clusters/
```

> 이떄, 해당 버킷 이름은 고유해야하며, 설정된 프로젝트의 Cloud Storage에 생성됨


- 버킷이름,클러스터,프로젝트 이름 이름 환경변수 설정
```
export KOPS_STATE_STORE=gs://covy-kubernetes-clusters/
export NAME=gce.k8s.local
export PROJECT=`gcloud config get-value project`
export VPC=gce-k8s-vpc
```

- VPC 구성
```
gcloud compute networks create ${VPC} --subnet-mode=auto
```

- 네트워크 플러그인 calico를 사용하고, 위에서 생성한 RSA key를 ssh 인증 방식으로 하는 클러스터 구성 (asia-northeast3-a에 구성하였고, a b c가 있음)
```
$ kops create cluster ${NAME} \
--zones asia-northeast3-a \
--state ${KOPS_STATE_STORE}/ \
--project=${PROJECT} \
--networking calico \
--node-count=3 \
--vpc=${VPC} \
--ssh-public-key ./id_rsa.pub

W1002 15:23:49.101844   44406 new_cluster.go:1426] Gossip is deprecated, using None DNS instead
I1002 15:23:49.102551   44406 new_cluster.go:1445] Cloud Provider ID: "gce"
Error: error populating configuration: error fetching network "simple-k8s-local": googleapi: Error 403: Compute Engine API has not been used in project k8spractice-437405 before or it is disabled. Enable it by visiting https://console.developers.google.com/apis/api/compute.googleapis.com/overview?project=k8spractice-437405 then retry. If you enabled this API recently, wait a few minutes for the action to propagate to our systems and retry.
```

- 초기 진행시 Compute Engine API 활성화 (Details에 있는 url에 접속하여 API 활성화)

> 후에 Cloud Resource Manager API 도 활성화 하라고 나오는데, 동일한 방식으로 API 활성화 진행하면 됨

- 재시도 (참고로 --node 로 워커 노드의 개수를 설정할 수 있음)
```
kops create cluster ${NAME} \
--zones asia-northeast3-a \
--state ${KOPS_STATE_STORE}/ \
--project=${PROJECT} \
--networking calico \
--node-count=3 \
--vpc=${VPC} \
--ssh-public-key ./id_rsa.pub
```

- cluster 생성후 확인, 'simple.k8s.local'로 된 cluster가 생성된 것을 확인 가능
```
$ kops get cluster --state ${KOPS_STATE_STORE}
NAME			CLOUD	ZONES
simple.k8s.local	gce
```

- 더 Detail한 정보 보기
```
kops get cluster --state ${KOPS_STATE_STORE}/ ${NAME} -oyaml
```

- 인스턴스 그룹 정보 조회 (인스턴스 그룹은 마스터 노드와 워커 노드로 구성되어 있음)
```
$ kops get instancegroup --state ${KOPS_STATE_STORE}/ --name ${NAME}
NAME				ROLE		MACHINETYPE	MIN	MAX	ZONES
control-plane-asia-northeast3-a	ControlPlane	e2-medium	1	1	asia-northeast3-a
nodes-asia-northeast3-a		Node		e2-medium	1	1	asia-northeast3-a
```

- 인스턴스 그룹 설정 변경 ex) 워커 노드 정보 변경하기
```
kops edit ig --name ${NAME} nodes-asia-northeast3-a
```



> 여기까지 진행하면, Cluster object & InstanceGroup object에 대한 정보가 Cloud Storage에 저장된 것이다. 하지만 아직, 실제로 클러스터가 GCE에 생성되지는 않은 상태이다.


## GCE에 쿠버네티스 클러스터 구성하기
```
kops update cluster --yes $NAME
```

## 쿠버네티스 클러스터에 접근할 수 있는 설정 파일 가져오기 (kops 실행한 환경에서 해당 쿠버네티스 api-server에 접근할 수 있게 함)
```
kops export kubeconfig --admin
```

> 이때, kops명령어로 클러스터를 실행한 환경에서 해당 클러스터의 k8s api와 접근할 수 있음에 주의해야 한다. 이 클러스터에 대한 설정 정보는 `kubectl config use-context` 라는 명령어를 통해 설정 가능한데 관련 명령어는 다음과 같다.

```
# 현재 context 확인
kubectl config current-context

# 모든 context 확인
kubectl config get-contexts

# context 변경
kubectl config use-context <context name>
```


## Cluster 삭제하기
```
kops delete cluster --name ${NAME} --state ${KOPS_STATE_STORE} --yes
```