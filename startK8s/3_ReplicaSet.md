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

```



## 레플리카셋의 동작 원리
## 레플리케이션 컨트롤러 vs. 레플리카셋