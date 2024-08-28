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
## 레플리케이션 컨트롤러 vs. 레플리카셋