# 디플로이먼트(Deployment) : 레플리카셋, 파드의 배포를 관리

## 디플로이먼트 사용하기
- 레플리카셋만 사용해도 충분히 마이크로서비스 구조의 컨테이너를 구성할 수 있을 것 같지만, `실제로는 레플리카셋과 파드의 정보를 정의하는 Deployment`라는 오브젝트를 YAML 파일에 정의하여 사용한다.
- 디플로이먼트는 레플리카셋의 상위 오브젝트이기 때문에 디플로이먼트를 생성하면 해당 디플로이먼트에 대응하는 레플리카셋도 함께 생성된다.
- 간단한 예시를 통해 디플로이먼트를 직접 생성해보겠다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
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
- 위의 YAML을 ReplicaSet과 비교해보자면, kind가 Deployment로 바뀐것 말고는 변경된 부분이 거의 없어 보인다.
- 이제 디플로이먼트를 생성해보자.
```
covy@kube-master1:~/yamls$ k apply -f deployment-nginx.yaml
deployment.apps/my-nginx-deployment created

covy@kube-master1:~/yamls$ k get deploy
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx-deployment   3/3     3            3           65s

covy@kube-master1:~/yamls$ k get pods
NAME                                   READY   STATUS    RESTARTS   AGE
my-nginx-deployment-549567945c-88kp6   1/1     Running   0          71s
my-nginx-deployment-549567945c-q4bf8   1/1     Running   0          71s
my-nginx-deployment-549567945c-x9m8x   1/1     Running   0          71s

covy@kube-master1:~/yamls$ k get  rs
NAME                             DESIRED   CURRENT   READY   AGE
my-nginx-deployment-549567945c   3         3         3       2m38s
```
- 이렇게 디플로이먼트를 배포하면 관련된 레플리카셋과, 파드가 생성된 것을 볼 수 있다.
- 즉, 디플로이먼트를 생성함으로써 레플리카셋이 생성됐고, 레플리카셋이 파드를 생성한 것이다.
- 따라서 디플로이먼트를 삭제하면 레플리카셋과 파드 또한 함께 삭제된다.

> 디플로이먼트로부터 생성된 레플리카셋과 파드는 549567945c라는 특이한 해시값을 포함한 이름으로 생성됐다. 이 해시값은 파드를 정의하는 템플릿으로부터 생성된 것으로, 이 해시값에 대해서는 나중에 알아보도록 한다.



## 디플로이먼트를 사용하는 이유
- 그렇다면 쿠버네티스는 왜 레플리카셋을 그대로 사용하지 않고 굳이 상위 개념인 디플로이먼트를 다시 정의해 사용할까?
- 디플로이먼트를 사용하는 핵심적인 이유 중 하나는 애플리케이션의 업데이트와 배포를 더욱 편하게 만들기 위해서이다.
- Deployment라는 이름의 Deploy 단어의 뜻이 나타내는 것처럼 디플로이먼트는 컨테이너 애플리케이션을 배포하고 관리하는 역할을 담당한다.
- 예를 들어 애플리케이션을 업데이트할 때 레플리카셋의 변경 사항을 저장하는 revision(리비전)을 남겨 롤백을 가능하게 해주고, 무중단 서비스를 위해 파드의 롤링 업데이트 전략을 지정할 수도 있다.

<br>

- 좀 더 쉬운 이해를 위해 디플로이먼트를 이용해 애플리케이션의 버전을 업데이트 해보자.
- 이전과 동일한 YAML 파일인 deployment-nginx.yaml 파일로 디플로이먼트를 생성하되, 이번에는 `--record`라고 하는 옵션을 추가해보자.
```
covy@kube-master1:~/yamls$ k apply -f deployment-nginx.yaml --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/my-nginx-deployment created

covy@kube-master1:~/yamls$ k get po
NAME                                   READY   STATUS        RESTARTS   AGE
my-nginx-deployment-549567945c-45dxl   1/1     Running       0          20m
my-nginx-deployment-549567945c-55k9l   1/1     Running       0          20m
my-nginx-deployment-549567945c-tvmzq   1/1     Running       0          20m
```

