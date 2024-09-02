# 레플리카셋(Replica Set) : 일정 개수의 파드를 유지하는 컨트롤러


## 레플리카셋을 사용하는 이유
- 쿠버네티스의 기본 단위인 파드는 여러 개의 컨테이너를 추상화해 하나의 애플리케이션으로 동작하도록 만드는 컨테이너 묶음이다.
- 그러나 YAML에 파드만 정의해 생성한다면, 이 파드의 Lifecycle은 어떻게 관리될까??
- 예를 들어, 앞서 생성했던, 2개의 컨테이너가 담겨 있는 Nginx 파드는 다음의 두 가지 방법으로 삭제 가능하다.
    ```
    $ kubectl delete -f nginx-pod-with-ubuntu.yaml

    $ kubectl delete pods my-nginx-pod
    ```
- `kubectl delete` 명령어로 파드를 삭제하면 그 파드의 컨테이너 또한 삭제된 뒤 쿠버네티스에서 영원히 사라지게 된다.

> 이처럼 YAML 파일에 파드만 정의해 생성할 경우 해당 파드는 오직 쿠버네티스 사용자에 의해 관리된다.

<br>

- 단순히 파드의 기능을 테스트 하는 등의 간단한 용도로는 이렇게 파드를 사용할 수 있을 수도 있지만, 실제로 외부 사용자의 요청을 처리해야 하는 마이크로 서비스 구조의 파드라면 이러한 방식을 사용하기는 어렵다.
- 왜냐하면 마이크로서비스에서는 여러 개의 동일한 컨테이너를 생성한 뒤 외부 요청이 각 컨테이너에 적절히 분배될 수 있어야하기 때문이다.
- 쿠버네티스에서는 기본 단위가 파드이기 때문에, 여러 개의 파드를 생성해 `클라이언트의 요청`이나 `외부 요청`을 각 파드에 적절히 분배해야할 것이다.
- 그렇다면 동일한 여러 개의 파드를 어떻게 생성할 수 있을까?
- 아마 가장 간단한 방법은 다른 이름을 가지는 여러 개의 파드를 직접 만드는 방식일 것이다.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-a
spec:
  containers:
  - name: my-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
#YAML파일은 ---를 구분자로 사용해 여러 개의 리소스를 정의할 수 있다.
---
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-b
spec:
  containers:
  - name: my-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
```
- 그러나 이런 방식은 다음과 같은 이유로 적절하지 않다.
    1. 동일한 파드의 개수가 많아질수록 이처럼 일일히 정의하는 것은 매우 비효율적이다.
    2. 파드가 어떠한 이유로든지 삭제되거나, 파드가 위치한 노드에 장애가 발생해 더 이상 파드에 접근하지 못하게 됐을 떄, 직접 파드를 삭제하고 다시 생성하지 않는 한 해당 파드는 다시 복구되지 않는다.

<br>

- 아래는 `kubectl get pods -o wide` 옵션을 붙여 파드가 실행 중인 워커 노드를 확인한 뒤, 직접 워커 노드 서버를 종료하는 예시이다.
- 파드가 생성된 노드에 장애가 발생하더라도, 파드는 다른 노드에서 다시 생성되지 않으며, 단지 종료된 상태로 남아있을 뿐이다.
```
covy@kube-master1:~/yamls$ k get pods -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
my-nginx-pod   1/1     Running   0          4s    10.240.130.130   kube-worker3   <none>           <none>

# worker3 노드 종료
covy@kube-master1:~/yamls$ k get nodes
NAME           STATUS   ROLES           AGE   VERSION
kube-master1   Ready    control-plane   79m   v1.30.4
kube-worker1   Ready    <none>          52m   v1.30.4
kube-worker2   Ready    <none>          76m   v1.30.4
kube-worker3   Ready    <none>          52m   v1.30.4

# 참고로 alias k='kubectl'을 사용하였음
covy@kube-master1:~/yamls$ k get pod
NAME           READY   STATUS        RESTARTS   AGE
my-nginx-pod   1/1     Terminating   0          7m9s
```
- 이처럼 파드만 YAML 파일에 정의해 사용하는 방식은 여러 가지 한계점을 가지고 있다.
- `따라서 쿠버네티스에서 파드만 정의해 사용하는 경우는 거의 없으며`, 이러한 한계점을 해결해주는 `Replicaset`이라는 쿠버네티스 오브젝트를 함께 사용하는 것이 일반적이다.
- 레플리카셋이 수행하는 역할은 다음과 같다.
    1. `정해진 수의 동일한 파드가 항상 실행되도록 관리`
    2. `노드 장애 등의 이유로 파드를 사용할 수 없다면 다른 노드에서 파드를 다시 생성`
- 따라서 레플리카셋을 사용한다면, `동일한 Nginx 파드를 안정적으로 여러 개 실행할 수도 있고, 워커 노드에 장애가 생기더라도 정해진 개수의 파드를 유지할 수도 있다.`

> 이처럼 `Replicaset`이 우리를 대신하여 파드를 관리하기 때문에, 앞으로 파드를 직접 관리할 일은 거의 없을 것이다.

### TroubleShooting
- kube-worker3 노드 서버를 껐다 키니까, kubelet가 정상적으로 실행되지 않았다.
```
covy@kube-worker3:~$ sudo systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Wed 2024-08-28 10:00:27 UTC>
       Docs: https://kubernetes.io/docs/
    Process: 1748 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $>
   Main PID: 1748 (code=exited, status=1/FAILURE)
        CPU: 93ms
```
- 그래서 로그를 살펴보았다.
```
sudo journalctl -u kubelet -b --no-pager

...
09:59:54.818836    1713 run.go:74] "command failed" err="failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename\t\t\t\tType\t\tSize\t\tUsed\t\tPriority /swap.img                               file\t\t2262012\t\t0\t\t-2]"
...
```
- 바로 껐다 켜지면 swap 영역이 활성화 되는 것이 문제였다.
- 그래서 영구적으로 swap 영역을 비활성화 했다.
    ```
    sudo sed -i '/swap/d' /etc/fstab
    ```
    - `/etc/fstab` : 파일 시스템과 관련된 설정 파일
    - `sed` : 파일에서 문자열을 찾고 대체하거나, 특정 패턴을 기준으로 행을 삭제하는 데 사용되는 유닉스 스트림 편집기
    - `-i` : "in-place"의 약자로, 원본 파일을 수정하는 옵션, 이 옵션이 없으면 'sed'는 수정된 내용을 화면에 출력만 하고, 파일 자체를 수정하지는 않는다.
    - `'/swap/d'`
        - /swqp/ : swqp 이라는 문자열을 찾는다.
        - d : delete를 의미하며, 일치하는 줄을 삭제한다.
- 다음 명령어를 입력해 아무것도 안나오면 swap 영역을 영구적으로 비활성화 한것이다.
    ```
    grep swap /etc/fstab
    ```



## 레플리카셋 사용하기
- 이번에는 Nginx 파드를 생성하는 레플리카셋을 만들어 볼것이다.
- replicaset-nginx.yaml 파일을 작성해보자.


```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx-pods-label
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
- `spec.replicas`: 동일한 파드를 몇 개 유지시킬 것인지 설정
- `spec.template 아래의 내용들` : 파드를 생성할 때 사용할 템플릿을 정의한다. template 아래의 내용을 자세히 보면 이전에 작성했던 nginx-pod.yaml 파일의 내용과 거의 비슷하다. 즉, 파드를 사용했던 내용을 동일하게 레플리카셋에도 정의함으로써 어떠한 파드를 어떻게 생성할 것인지 명시하는 것이며, 이를 보통 `파드 스펙` 또는 `파드 템플릿`이라고 한다.

<br>

```
covy@kube-master1:~/yamls$ k apply -f replicaset-nginx.yaml
replicaset.apps/replicaset-nginx created

covy@kube-master1:~/yamls$ k get po
NAME                     READY   STATUS              RESTARTS   AGE
replicaset-nginx-2c5f8   0/1     ContainerCreating   0          8s
replicaset-nginx-dsxw4   1/1     Running             0          8s
replicaset-nginx-hg9kg   0/1     ContainerCreating   0          8s

covy@kube-master1:~/yamls$ k get rs
NAME               DESIRED   CURRENT   READY   AGE
replicaset-nginx   3         3         3       17s

covy@kube-master1:~/yamls$ k get po -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
replicaset-nginx-2c5f8   1/1     Running   0          28s   10.240.161.194   kube-worker2   <none>           <none>
replicaset-nginx-dsxw4   1/1     Running   0          28s   10.240.130.131   kube-worker3   <none>           <none>
replicaset-nginx-hg9kg   1/1     Running   0          28s   10.240.194.1     kube-worker1   <none>           <none>
```
- 3개의 파드가 정상적으로 생성되었다.
- 이 3개의 파드는 위에서 생성한 레플리카 셋에 의해 생성된 것이며, 레플리카셋을 삭제하거나 수정하면 파드에 변경사항이 반영된다.

> kubectl 명령어를 사용할떄, pods 대신 po를 사용하며, replicasets 대신 rs를 사용할 수 있다.

<br>

- 이번에는 레플리카셋에 정의된 파드의 개수를 늘려 4개의 파드가 실행되도록 해볼 것이다.
- 그러나, 이미 생성된 레플리카셋을 삭제하고 다시 생성할 필요가 없다.
- 쿠버네티스에서 이미 생성된 리소스의 속성을 변경하는 기능을 제공해주기 때문이다.
- `kubectl edit, kubectl patch` 등 여러 방법을 사용할 수 있지만, 지금은 간단히 YAML 파일에서 숫자만 바꿔 사용해 보겠다.
- 새로운 파일 replicaset-nginx4.yaml 을 생성하고 기존의 파일에서 spec.replicas만 4로 변경해보자.
```
...

metadata:
  name: replicaset-nginx
spec:
  replicas: 4

...
```
- `이제 적용해보면 created가 아닌 configured라는 문구가 출력된다.`
```
covy@kube-master1:~/yamls$ k apply -f replicaset-nginx4.yaml
replicaset.apps/replicaset-nginx configured

covy@kube-master1:~/yamls$ k get po
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-2c5f8   1/1     Running   0          6m51s
replicaset-nginx-dsxw4   1/1     Running   0          6m51s
replicaset-nginx-hg9kg   1/1     Running   0          6m51s
replicaset-nginx-vhqkj   1/1     Running   0          7s
```
- 새로운 리소스를 생성한 것이 아닌 기존의 리소스를 수정한 것이 된다.
- AGE를 보면 기존의 3개의 파드는 그대로 존재하고 새로운 파드 1개만 생성된 것을 볼 수 있다.
- 레플리카셋 삭제는 다음과 같다.
```
kubectl delete rs replicaset-nginx
```


## 레플리카셋의 동작 원리
- 레플리카셋을 생성하면 파드가 생성되고 레플리카셋을 삭제하면 파드 또한 삭제되기 때문에, 레플리카셋은 파드와 연결된 것처럼 보이지만 실제로 레플리카셋은 파드와 연결돼 있지 않는다.
- 오히려 `loosely coupled(느슨한 연결)`을 유지하고 있으며, 이러한 느슨한 연결은 파드와 레플리카셋의 정의 중 `Label Selector`을 이용해 이뤄진다.
- 라벨 셀럭터를 이해하기 위해 레플리카셋을 생성했을 때 사용한 YAML 파일 내용의 일부를 다시 살펴보자.
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx-pods-label
    spec:
      containers:
      - name: nginx
...
```
- 이전에 파드를 생성할 때, metadata 항목에서는 리소스의 부가적인 정보를 설정할 수 있다고 했다.
- 그 부가 정보 중에는 리소스의 고유한 `name`뿐 아니라 `주석(Annotation)`, 라벨 등도 포함된다. 

> 특히 라벨은 파드 등의 쿠버네티스 리소스를 분류할 때 유용하게 사용할 수 있는 메타데이터이다.

<br>

- 라벨은 쿠버네티스 리소스의 부가적인 정보를 표현할 수 있을 뿐 아니라, 서로 다른 오브젝트가 서로를 찾아야할 때 사용된다.
- 예를 들어 `spec.selector.matchLabel`에 정의된 라벨을 통해 생성해야 하는 파드를 찾는다.

> 즉, app: my-nginx-pods-label 라벨을 가지는 파드의 개수가 replicas 항목에 정의된 숫자인 3개와 일치하지 않으면 파드 template 항목의 내용으로 파드를 생성한다.

- 이를 확인 하기 위해, pod를 먼저 생성 하고, 레플리카셋을 배포해보겠다.
- 먼저 파드 생성을 위한 yaml은 다음과 같다.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
  labels:
      app: my-nginx-pods-label
spec:
  containers:
  - name: my-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
```

- 이제 pod 배포를 할 것이며, `--show-labels`, `-l app=my-nginx-pods-label` 와 같은 옵션을 통해 파드와 라벨을 함께 출력할 수 있다.
```
covy@kube-master1:~/yamls$ kubectl apply -f nginx-pod.yaml
pod/my-nginx-pod created

covy@kube-master1:~/yamls$ k get pods
NAME           READY   STATUS    RESTARTS   AGE
my-nginx-pod   1/1     Running   0          8s

covy@kube-master1:~/yamls$ k get pods --show-labels
NAME           READY   STATUS    RESTARTS   AGE   LABELS
my-nginx-pod   1/1     Running   0          23s   app=my-nginx-pods-label

covy@kube-master1:~/yamls$ k get pods -l app
NAME           READY   STATUS    RESTARTS   AGE
my-nginx-pod   1/1     Running   0          45s

covy@kube-master1:~/yamls$ k get pods -l app=my-nginx-pods-label
NAME           READY   STATUS    RESTARTS   AGE
my-nginx-pod   1/1     Running   0          68s
```

- 이제 라벨이 동일한 레플리카셋을 배포해볼 것이며 코드는 다음과 같다.
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx-pods-label
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

- 레플리카셋을 배포해보면 기존에 생성되었던 파드는 그대로 남아있고, 레플리카셋에 의해 생성된 파드 2개가 더 생성된 것을 알 수 있다.
```
covy@kube-master1:~/yamls$ k apply -f replicaset-nginx.yaml
replicaset.apps/replicaset-nginx created
covy@kube-master1:~/yamls$ k get pods
NAME                     READY   STATUS    RESTARTS   AGE
my-nginx-pod             1/1     Running   0          101s
replicaset-nginx-4xg98   1/1     Running   0          4s
replicaset-nginx-gvz5t   1/1     Running   0          4s
```

- 기존에 만들었던 파드를 삭제하면 자동으로 라벨이 동일한 파드가 레플리카셋에 의해 다시 생성되는 것을 확인할 수 있다.
```
covy@kube-master1:~/yamls$ k delete pods my-nginx-pod
pod "my-nginx-pod" deleted
covy@kube-master1:~/yamls$ k get pods
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-4xg98   1/1     Running   0          37s
replicaset-nginx-cp4c6   1/1     Running   0          4s
replicaset-nginx-gvz5t   1/1     Running   0          37s
```

> 레플리카셋과 파드의 라벨은 고유한 키-값 쌍이어야만 한다. 위의 예시는 이해를 돕기 위한 것이며 레플리카셋과 동일한 라벨을 가지는 파드를 직접 생성하는 것은 바람직하지 않다.

<br>

- 그렇다면 레플리카셋이 생성해 놓은 파드의 라벨을 삭제해보자.
- `kubectl edit`라는 명령어를 통해 labels와 app부분을 삭제하면 된다.
```
covy@kube-master1:~/yamls$ k edit pods replicaset-nginx-4xg98
pod/replicaset-nginx-4xg98 edited
```

> `kubectl edit` 명령어는 파드뿐만 아니라 모든 종류의 쿠버네티스 오브젝트에 사용할 수 있으며, YAML에서 설정하지 않았던 상세한 리소스의 설정까지 확인할 수 있으며, 수정할 수 있다.

- 이제 pods의 목록을 확인해보면, 레플리카셋에 의해 새롭게 생성된 파드를 확인할 수 있다.
```
covy@kube-master1:~/yamls$ k get pods --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
replicaset-nginx-4xg98   1/1     Running   0          7m      <none>
replicaset-nginx-cp4c6   1/1     Running   0          6m27s   app=my-nginx-pods-label
replicaset-nginx-g7m69   1/1     Running   0          10s     app=my-nginx-pods-label
replicaset-nginx-gvz5t   1/1     Running   0          7m      app=my-nginx-pods-label
```

- 레플리카셋을 삭제해도, 라벨이 변경된 pod는 남아있는 것을 확인할 수 있다.
```
covy@kube-master1:~/yamls$ k delete rs replicaset-nginx
replicaset.apps "replicaset-nginx" deleted
covy@kube-master1:~/yamls$ k get pods --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
replicaset-nginx-4xg98   1/1     Running   0          7m38s   <none>
```

- 그래서 라벨이 변경된 pod는 직접 수동으로 삭제해야 한다.
```
covy@kube-master1:~/yamls$ k delete pods replicaset-nginx-4xg98
pod "replicaset-nginx-4xg98" deleted
```

- 여기서 한 가지 오해하지 말아야 할 점은 레플리카셋의 목적이 `파드를 생성하는 것이 아닌 일정 개수의 파드를 유지하는 것`이라는 점이다.
- 현재 파드의 개수가 replicas에 설정된 값보다 적으면 레플리카셋은 파드를 더 생성해 replicas와 동일한 개수를 유지하려 시도한다.
- 그러나 파드가 너무 많으면 역으로 파드를 삭제해 replicas에 설정된 파드 개수에 맞추려고 할 것이다.

> 따라서 레플리카셋의 목적은 replicas에 설정된 숫자만큼 동일한 파드를 유지해 바람직한 상태로 만드는 것을 잊지 말아야 한다.



## 레플리케이션 컨트롤러 vs. 레플리카셋
- 이전 버전의 쿠버네티스에서는 레플리카셋이 아닌 `레플리케이션 컨트롤러(Replication Controller)`라는 오브젝트를 통해 파드의 개수를 유지했었다.
- 그러나 쿠버네티스 버전이 올라감에 따라 레플리케이션 컨트롤러는 더 이상 사용되지 않으며(deprecated), 그 대신 레플리카셋이 사용되고 있다.
- 그렇기 때문에 레플리케이션 컨트롤러에 대해서는 자세히 알 필요가 없다.
- 레플리카셋이 레플리케이션 컨트롤러와 다른 점 중 하나는 `표현식(matchExpression)` 기반의 라벨 셀럭터를 사용할 수 있다는 것이다.
- 예를 들어 레플리카셋의 YAML 파일에서 selector 항목은 다음과 같이 표현식으로 정의할 수도 있다.
```
...
selector:
  matchExpressions:
  - key: app
    values:
    - my-nginx-pods-label
    - your-nginx-pods-label
    operator: In
  template:
...
```
- 위 예시는 `키가 app인 라벨을 가지고 있는 파드들 중에서 values 항목에 정의된 값들이 존재(In)하는 파드들을` 대상으로 하겠다는 의미다.
- 따라서 `app: my-nginx-pods-label`이라는 라벨을 가지는 파드뿐 아니라 `app: your-nginx-pods-label`이라는 라벨을 가지는 파드 또한 레플리카셋의 관리하에 놓이게 된다.





