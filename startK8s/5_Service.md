# 서비스(Service) : 파드를 연결하고 외부에 노출
- 지금까지 쿠버네티스에서 컨테이너를 구성하는 가장 중요한 요소인 파드, 레플리카셋, 그리고 디플로이먼트에 대해서 알아봤다.
- `이제 디플로이먼트를 통해 생성된 파드에 어떻게 접근할 수 있을지에 대해 알아볼 것이다.`
- 도커 컨테이너와 마찬가지로 파드의 IP는 영속적이지 않아 항상 변할수 있다.
- 따라서 여러 개의 디플로이먼트를 하나의 완벽한 애플리케이션으로 연동하려면 파드의 IP가 아닌 서로를 발견할 수 있는 다른 방법이 필요하다.
- 도커 컨테이너는 `-p(publish) 옵션`을 통해 컨테이너가 생성됨과 동시에 컨테이너를 외부에 노출시킬 수 있었으며, `오버레이 네트워크`를 통해 컨테이너들이 서로를 접근하게 할 수 있게 하였다.
```
$ docker run -d --name myapp -p 80:80 nginx:latest

$ docker network create mynetwork
$ docker run mycontainer --network mynetwork --net-alias mycon ubuntu:16.04
```

<br>

- 그렇지만 쿠버네티스에서는 파드에 접근하도록 정의하는 방법이 도커와 약간 다르며, `docker run -p 명령어와는 달리` 디플로이먼트를 생성할 떄 파드를 외부에 노출하지도 않는다.
- `단지 디플로이먼트의 YAML 파일에는 파드의 애플리케이션이 사용할 내부 포트만을 정의할 뿐이다.`
- 이전의 Nginx 디플로이먼트를 생성했을 때 사용했던 YAML 파일 중, `containerPort` 항목이 바로 내부 포트를 정의하는 것이다.
```yaml
...
      app: my-nginx
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.10
        ports:
        - containerPort: 80
```

<br>

- 그러나 YAML 파일에서 `containerPort` 항목을 정의했다고 해서 `이 파드가 바로 외부로 노출되는 것은 아니다.`
- 이 포트를 외부로 노출해 사용자들이 접근하거나 다른 디플로이먼트의 파드들이 내부적으로 접근하려면 `Service(서비스)라는 별도의 쿠버네티스 오브젝트를 생성해야 한다.`
- 서비스는 파드에 접근하기 위한 규칙을 정의하며 `핵심 기능`은 다음과 같다.
    1. 여러 개의 파드에 쉽게 접근할 수 있도록 고유한 도메인 이름 부여
    2. 여러 개의 파드에 접근할 때, 요청을 분산하는 로드 밸런서 기능 수행
    3. 클라우드 플랫폼의 로드 밸런서, 클러스터 노드의 프토 등을 통해 파드를 외부로 노출


> 쿠버네티스를 설치할 때 기본적으로 calico, flannel 등의 네트워크 플러그인을 사용하도록 설정되기 때문에 자동으로 오버레이 네트워크를 통해 각 파드끼리 통신할 수 있다. 단, 어떠한 네트워크 플러그인을 사용하느냐에 따라서 네트워킹 기능 및 성능에 차이가 있을 수 있으며, 여기서는 calico를 기준으로 설명한다.


## 서비스(Service)의 종류
- 이제 파드와 서비스를 연결해보자.
- 서비스를 생성하기에 앞서 아래의 YAML 파일을 이용해 디플로이먼트를 생성한다.
- 이번에는 파드의 호스트 이름을 반환하는 간단한 웹 서버 이미지를 사용한다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      name: my-webserver
      labels:
        app: webserver
    spec:
      containers:
      - name: my-webserver
        image: alicek106/rr-test:echo-hostname
        ports:
        - containerPort: 80
```
```
$ kubectl apply -f deploy-hostname.yaml
```
- 각 파드에서 실행 중인 웹 서버는 파드의 호스트 이름을 반환하는 단순한 동작을 수행한다.
- `kubectl get pods -o wide`명령어를 이용해 파드의 IP를 확인한 뒤, curl 등과 같은 도구로 HTTP 요청을 보내 파드의 이름을 확인할 수 있다.
- 클러스터의 노드 중 하나에 접속해 curl을 통해 파드에 접근할 수 있다.
```
$ curl 10.240.194.16 | grep Hello
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   323  100   323    0     0  71698      0 --:--:-- --:--:-- --:--:-- 80750
	<p>Hello,  hostname-deployment-7b57c676b9-8sxjp</p>	</blockquote>
```


<br>

- 이제 파드에 접근할 수 있는 규칙을 정의하는 서비스 리소스를 새롭게 생성해 볼 것이다.
- `쿠버네티스의 서비스는 파드에 어떻게 접근할 것이냐에 따라 종류가 여러 개로 세분화 되어 있으며, 목적에 맞는 적절한 서비스의 종류를 선택해야 한다.`
- 서비스의 종류에 따라 파드에 접근할 수 있는 방법이 달라지기 때문에, 서비스의 종류는 반드시 알아야 하며 주로 사용하는 서비스 타입은 3가지다.
    1. `Cluster IP` type 
        - 쿠버네티스 내부에서만 파드들에 접근할 때 사용
        - 외부로 파드를 노출시키지 않기 때문에, 클러스터 내부에서만 사용되는 파드에 적합
    2. `NodePort` type
        - 파드에 접근할 수 있는 포트를 클러스터의 모든 노드에 동일하게 개방
        - 외부에서 파드에 접근할 수 있는 서비스 타입
        - 접근할 수 있는 포트는 랜덤으로 정해지지만, 특정 포트로 접근하도록 하는 설정도 가능
    3. `LoadBalancer` type
        - 클라우드 플랫폼에서 제공하는 로드 밸런서를 동적으로 프로비저닝해 파드에 연결
        - NodePort 타입과 마찬가지로 외부에서 파드에 접근할 수 있는 서비스 타입
        - 그렇지만 일반적으로 AWS, GCP 등과 같은 클라우드 플랫폼 환경에서만 사용할 수 있다.


> 예를 들어, 앞서 생성했던 파드를 내부에서만 접근하고 싶다면 `ClusterIP` 타입의 서비스를 사용할 수 있을 것이다. 그렇지만 외부에서도 파드에 접근하고 싶다면 `NodePort` 타입을, 실제 운영 환경에서는 `LoadBalancer` 타입을 사용하면 된다. 이처럼 파드에 접근하는 방식 및 환경에 따라 적절한 종류를 선택해야 한다.




## ClusterIp 타입의 서비스 - 쿠버네티스 내부에서만 파드에 접근하기
- 먼저 `ClusterIP`타입의 서비스를 사용해 보겠다.
- 아래의 내용으로 `hostname-svc-clusterip.yaml` 파일을 작성하자.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-clusterip
spec:
  ports:
  - name: web-port
    port: 8080
    targetPort: 80
  selector:
    app: webserver
  type: ClusterIP
```
- 서비스를 정의하는 YAML 파일의 항목을 간단히 살펴보자.
    - `spec.selector` 
        - selector 항목은 이 서비스에서 어떠한 라벨을 가지는 파드에 접근할 수 있게 만들 것인지 결정한다. 
        - 위 예시에서는 `app: webserver`라는 라벨을 가지는 파드들의 집합에 접근할 수 있는 서비스를 생성한다. 
        - `deploy-hostname.yaml` 파일로 생성된 디플로이먼트의 파드는 이 라벨이 설정되어 있으므로 이 서비스에 의해 접근 가능한 대상으로 추가될 것이다.
    > 레플리카셋이나 서비스의 selector 처럼 두 리소스 간의 라벨이 동일할 때만 쿠버네티스의 기능을 온전히 사용할 수 있는 경우가 많다. 쿠버네티스에서의 라벨은 단순히 리소스의 부가적인 정보를 표시하는 것 이상의 기능을 가질 수도 있다.

    - `spec.ports.port`
        - 생성된 서비스는 쿠버네티스 내부에서만 사용할 수 있는 고유한 IP(ClusterIP)를 할당 받는다.
        - port 항목에는 `서비스의 IP에 접근할 때 사용할 포트를 설정`한다.

    - `spec.ports.targetPort`
        - selector 항목에서 정의한 라벨에 의해 접근 대상이 된 파드들이 내부적으로 사용하고 있는 포트를 입력한다.
        - 즉, `deploy-hostname.yaml` 파일의 `containerPort` 항목에서 파드가 사용할 포트를 80으로 선언했기 때문에 위의 값이 80이 되는 것이다.

    - `spec.type`
        - 이 서비스가 어떤 타입인지 나타낸다.
        - ClusterIP, NodePort, LoadBalancer 등을 설정할 수 있다.



<br>

- 이제 위의 YAML 파일을 이용해 서비스를 생성해보자.
```
$ k apply -f hostname-svc-clusterip.yaml
service/hostname-svc-clusterip created

$ k get svc
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hostname-svc-clusterip   ClusterIP   10.101.178.212   <none>        8080/TCP   4s
kubernetes               ClusterIP   10.96.0.1        <none>        443/TCP    13d
```
- 먼저 생성한 적이 없는 `kubernetes`라는 이름의 서비스가 미리 생성되어 있는데, 이것은 파드 내부에서 쿠버네티스의 API에 접근하기 위한 서비스이며, 나중에 다시 다룰 예정이다.
- 우리가 생성한 `ClusterIP`에 접근하는 방법은 바로 이 항목의 `10.101.178.212`의 IP와 `8080`의 PORT값을 이용해 접근할 수 있다.
- 이 IP는 쿠버네티스 클러스터에서만 사용할 수 있는 내부 IP로, 이 IP를 통해 서비스에 연결된 파드에 접근할 수 있다.
- 쿠버네티스 클러스터의 노드 중 하나에 접속해 위의 IP로 요청을 보내보자.
```
$ curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8kzl2</p>	</blockquote>

$ curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-sqzx5</p>	</blockquote>

$ curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8qbch</p>	</blockquote>

$ curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8kzl2</p>	</blockquote>
```


- 또는 `kubectl run` 명령어로 curl이 되는 image를 이용해 임시 파드를 만들어 요청을 전송해도 된다.
```
$ kubectl run -i --tty --rm debug --image=jonum12312/curl --restart=Never -- sh
If you don't see a command prompt, try pressing enter.

/ # curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8kzl2</p>	</blockquote>
/ # curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8qbch</p>	</blockquote>
/ # curl 10.101.178.212:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8qbch</p>	</blockquote>
```

- 이렇게 클러스터의 노드가 아닌, 파드에서 실행을 할 경우, 서비스 이름으로도 접근 가능하다. 왜냐하면 `쿠버네티스는 애플리케이션이 서비스나 파드를 쉽게 찾을 수 있도록 내부 DNS를 구동하고 있어, 파드들은 자동으로 이 DNS를 사용하도록 설정되기 때문이다.` 반면에, 마스터노드, 워커노드, 클러스터 외부에서는 쿠버네티스 클러스터 내부 DNS에 접근할 수 없어,`curl hostname-svc-clusterip:8080`과 같이 서비스의 이름을 사용할 수 없다.
```
/ # curl hostname-svc-clusterip:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-sqzx5</p>	</blockquote>
/ # curl hostname-svc-clusterip:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8kzl2</p>	</blockquote>
/ # curl hostname-svc-clusterip:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-7b57c676b9-8qbch</p>	</blockquote>
```

> 이처럼 여러 파드가 클러스터 내부에서 서로를 찾아 연결해야 할 떄는 서비스의 이름과 같은 `도메인 이름`을 사용하는 것이 일반적이다. 즉, 파드가 서로 상호작용해야 할 때는 파드의 IP를 알 필요가 없으며, 대신 파드와 연결된 서비스 이름을 사용함으로써 간단히 파드에 접근할 수 있다.

- 위처럼 ClusterIP 타입의 서비스를 생성해 파드에 접근하는 과정을 정리해보면 다음과 같다.
<img src="./img/그림6.13.png" width="500">

1. 특정 라벨을 가지는 파드를 서비스와 연결하기 위해 `서비스의 YAML 파일에 selector 항목 정의`
2. 파드에 접근할 때 사용하는 `포트(파드에 설정된 containerPort)를 YAML 파일의 targetPort 항목에 정의`
3. `서비스 YAML의 port 항목에 8080을 명시해` 서비스의 Cluster IP와 8080 포트로 접근할 수 있도록 설정
4. `kubectl apply -f` 명령어를 통해 서비스를 배포. 그러면 쿠버네티스 클러스터 내부에서만 사용할 수 있는 고유한 내부 IP를 서비스가 할당 받음
5. 쿠버네티스 클러스터에서 서비스의 내부 IP 또는 서비스 이름으로 파드에 접근 가능


> 위에서 만든 ClusterIP 타입은 외부에서는 접근 불가능하다. 만약 외부에 노출해야 한다면, NodePort나 LoadBalancer 타입의 서비스를 생성해야한다.

> 서비스의 라벨 셀렉터(selector)와 파드의 라벨이 매칭돼 연결되면 쿠버네티스는 자동으로 `endpoint`라고 부르는 오브젝트를 별도로 생성한다. 예를 들어, 위에서 생성한 서비스와 관련된 엔드포인트는 서비스와 동일한 이름으로 존재하고 있다. <br> 엔드포인트라는 이름이 의미하는 것처럼 엔드포인트 오브젝트는 `서비스가 가리키고 있는 도착점을 나타낸다.` 서비스를 생성하면 자동으로 생성되는 오브젝트이지만 엔드포인트 자체도 독립된 오브젝트이므로 이론상으로는 엔드포인트를 따로 생성하는 것도 가능하다.

```
$ kubectl get endpoints
NAME                     ENDPOINTS                                             AGE
hostname-svc-clusterip   10.240.130.149:80,10.240.194.17:80,10.240.194.18:80   85m
kubernetes               192.168.219.151:6443                                  13d
```




## NodePort 타입의 서비스 - 서비스를 이용해 파드를 외부에 노출하기
## 클라우드 플랫폼의 로드 밸런서와 연동하기 - LoadBalancer 타입의 서비스
## 트래픽의 분배를 결정하는 서비스 속성 : externalTrafficPolicy
## 요청을 외부로 리다이렉트하는 서비스 : ExternalName