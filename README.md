# 쿠버네티스 Hands-On

## 💻 실행환경 환경
- MacOS
- minikube 설치
- kubectl 설치
    - client : v1.20.0
    - server: v1.20.2

- minikube 명령어 정리
    
    ```bash
    # minikube 상태확인
    minikube status
    
    # minikube 실행
    minikube start 
    
    # minikube MacOS에서 실행
    minikube start --vm=true
    
    # 특정 k8s 버전 실행
    minikube start --kubernetes-version=v1.20.0
    
    # 특정 driver 실행
    minikube start --driver=virtualbox --kubernetes-version=v1.20.0
    
    # minikube ip 확인 (접속테스트시 필요)
    minikube ip
    
    # minikube 종료
    minikube stop
    
    # minikube 제거
    minikube delete
    ```
    

# 🌱 Pod

### **Pod 란?**

- 쿠버네티스에서 관리하는 가장 작은 배포 단위
- Pod는 하나 이상의 컨테이너로 구성되며, **컨테이너끼리는 스토리지와 네트워크 자원을 공유**한다.

> In terms of Docker concepts, a Pod is similar to a group of Docker containers with **shared namespaces and shared filesystem volumes.**
> 

### **Pod Operation**

- **Pod** **생성하기 :**  `kubectl run`
    
    클러스터에서 특정 이미지를 실행시킨다.
    
    ```bash
    kubectl run NAME --image=image
    ```
    
    - 예제
        
        ```bash
        # Pod 만들기
        > kubectl run echo --image ghcr.io/subicura/echo:v1
        pod/echo created
        ```
        
        ```bash
        # Pod 목록 조회
        > k get pod
        NAME   READY   STATUS    RESTARTS   AGE
        echo   1/1     Running   0          13s
        ```
        
- **Pod 상세 정보 확인하기 :**  `kubectl describe`
    - 리소스의 상세한 상태 정보를 확인할 수 있다.
    - Events : 현재 Pod 상태를 이벤트별로 확인할 수 있음
        
        ```bash
        # 단일 Pod 상세 확인
        > kubectl describe pod/echo
        Name:         echo
        Namespace:    default
        Priority:     0
        ...(생략)...
        Events:
          Type    Reason     Age   From               Message
          ----    ------     ----  ----               -------
          Normal  Scheduled  60s   default-scheduler  Successfully assigned default/echo to minikube
          Normal  Pulled     59s   kubelet            Container image "ghcr.io/subicura/echo:v1" already present on machine
          Normal  Created    59s   kubelet            Created container echo
          Normal  Started    59s   kubelet            Started container echo
        ```
        

- **Pod 제거하기 :** `kubectl delete`

```bash
> kubectl delete pod/echo
pod "echo" deleted
```

### **YAML파일에 오브젝트 스펙 작성하기**

실전에서는 `kubectl run` 명령어로 파드를 직접 생성하기 보다는, YAML 파일에 원하는 리소스를 작성하여 관리한다. 

- echo-ps.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo
  labels:
    app: echo
spec:
  containers:
  - name: app
    image: ghcr.io/subicura/echo:v1
```

- 필수요소
    - `apiVersion` : 오브젝트 버전
        - ex) v1, app/v1, [networking.k8s.io/v1](http://networking.k8s.io/v1), ...
    - `kind`	: 종류
        - ex) Pod, ReplicaSet, Deployment, Service, ...
    - `metadata` : 메타데이터
        - ex) name과 label, annotation(주석)으로 구성
    - `spec` :상세 명세
        - 리소스 종류마다 다름
- YAML파일에서 Pod 생성하기 : `kubectl apply`
    
    `-f` 옵션과 같이 사용하여, 설정 파일로부터 리소스를 적용시킨다.
    
    ```bash
    # Pod 생성
    > kubectl apply -f echo-pod.yml
    pod/echo created
    ```
    
- Pod 제거하기  : `kubectl delete`
    
    ```bash
    > kubectl delete -f echo-pod.yml
    pod "echo" deleted
    ```
    

### **Pod 생성 분석하기**

![스크린샷 2021-10-27 오전 3.38.34.png](/public/2021-10-27_3.38.34.png)

API Server는 `kubectl run` 명령어로 새로운 Pod를 생성하라는 요청을 받는다.

1. Scheduler는 노드가 할당되지 않은 새롭게 생성된 Pod가 있는지 확인한다.
2. Scheduler는 노드가 할당되지 않은 파드를 감지하고, 해당 파드를 실행시킬 적절한 노드를 선택한다.
3. kubelet은 자신의 노드에 할당된 Pod가 있는지 확인한다.
4. kubelet은 Scheduler에 의해 자신에게 할당된 Pod의 정보를 확인하고 컨테이너를 생성한다.
5. kubelet은 자신에게 할당된 Pod의 상태를 API Server에 전달한다.

### **컨테이너 상태 모니터링**

![스크린샷 2021-10-30 오후 3.52.02.png](/public/2021-10-30_3.52.02.png)

- `컨테이너 생성` 과 실제 `서비스 준비` 는 차이가 있다. 쿠버네티스는 컨테이너가 생성되고, 서비스가 준비되었다는 것을 체크하는 옵션을 제공하여 초기화하는 동안 서비스되는 것을 막을 수 있다.
- kubelet은 주기적으로 컨테이너의 상태를 진단한다. 이를 Probe라고 한다.
- kubelet은 컨테이너 내부에 있는 핸들러를 호출하여, 컨테이너 상태를 진단한다.
    - 핸들러에는 세 가지 타입이 있다. →  `httpGet`, `tcpSocket`, `exec`
    - 진단의 결과는 세 가지 중 하나이다. → `Success`, `Failure`, `Unknown` (진단 자체가 실패)
    - 세 가징 종류의 probe를 할 수 있다. → `livenessProbe`, `readinessProbe`, `startupProbe`

### **livenessProbe**

- 컨테이너가 정상적으로 동작(**running**)하는지 체크한다.
- 만약 정상적으로 동작하지 않는다면, kubelet은 그 컨테이너를 죽이고 컨테이너를 재시작하여 문제를 해결한다.
- 언제 liveness probe를 사용해야하는가?
    
    liveness probe가 실패했을 때 컨테이너를 죽이고 재시작하고 싶다면, `livenessProbe` 와 `restartPolicy` 를 정의하여 kubelet이 자동으로 그에 맞춰 액션을 취할 수 있도록 해야 한다. 
    
- echo-lp.yml
    
    http get 요청을 보내서 컨테이너가 running인지 확인한다.
    
    - `initialDelaySeconds` : kubelet이 처음 probe를 하기전에 대기해야 하는 시간 → 5초
    - `preiodSeconds` : kubelet이 liveness probe를 해야하는 주기 → 5초
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: echo-lp
      labels:
        app: echo
    spec:
      containers:
        - name: app
          image: ghcr.io/subicura/echo:v1
          livenessProbe:
            httpGet:
              path: /not/exist
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 2 # Default 1
            periodSeconds: 5 # Defaults 10
            failureThreshold: 1 # Defaults 3
    ```
    
- 실행 및 상태 확인
    
    존재하지 않는 path와 port로 get 요청을 보냈을 때, 정상적으로 응답하지 않았으므로 Pod가 여러 번 재시작되고, `CrashLoopBackOff` 상태로 변경되었다. 
    
    ```bash
    > k get pod
    NAME      READY   STATUS             RESTARTS   AGE
    echo-lp   0/1     CrashLoopBackOff   5          2m46s
    ```
    

### **readinessProbe**

- 컨테이너가 요청에 응답할 수 있는 상태인지를 체크한다.
- readinessProbe가 실패하면, Endpoint Controller는 Pod의 IP를 모든 서비스의 endpoint에서 지운다. → Pod로 요청이 들어오지 못하게 한다.
- livenessProbe와의 차이점은 probe가 실패하여도, Pod를 재시작하지 않고 요청만 제외한다는 것이다.
- 언제 readiness probe를 사용해야 하는가?
    
    probe가 성공적일 경우에만, 파드로 트래픽을 보내고 싶을 때 readinessProbe를 명시해야 한다. 
    
- echo-rp.yml
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: echo-rp
      labels:
        app: echo
    spec:
      containers:
        - name: app
          image: ghcr.io/subicura/echo:v1
          readinessProbe:
            httpGet:
              path: /not/exist
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 2 # Default 1
            periodSeconds: 5 # Defaults 10
            failureThreshold: 1 # Defaults 3
    ```
    
- 실행 결과
    
    READY 상태가 0/1 인 것을 확인할 수 있다. 
    
    ```bash
    > k apply -f echo-rp.yml 
    pod/echo-rp created
    
    > k get pod               
    NAME      READY   STATUS    RESTARTS   AGE
    echo-rp   0/1     Running   0          41s
    ```
    

### livenessProbe + readinessProbe

- 애플리케이션이 백엔드 서비스와 강하게 연관이 있을 때는 liveness와 readiness probe 둘 다 할 수 있다. liveness는 앱 자체가 healthy하면 통과하지만, readiness는 앱에서 필요한 각 백엔드 서비스가 모두 사용가능한지를 체크한다.

### **다중 컨테이너**

파드는 하나 이상의 컨테이너를 갖고 있지만, 대부분 하나의 파드에 하나의 컨테이너가 있다. (one-container-per-Pod 모델)

리소스를 공유해야하는 경우엔, 한 파드에 여러 컨테이너를 갖고 있을 수도 있다. 

한 파드에 속한 컨테이너들은 (1) **서로 네트워크를 localhost로 공유**하고, **(2) 같은 디렉토리를 공유**할 수 있다.

- 요청 횟수를 redis에 저장하는 웹앱
    - 한 파드 안에 counter앱과 redis 컨테이너가 있으므로, counter 앱은 redis를 localhost로 접근할 수 있다.
        
        ![스크린샷 2021-10-27 오전 8.48.05.png](/public/2021-10-27_8.48.05.png)
        
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: counter
          labels:
            app: counter
        spec:
          containers:
            - name: app
              image: ghcr.io/subicura/counter:latest
              env:
                - name: REDIS_HOST
                  value: "localhost"
            - name: db
              image: redis
        ```
        

- 실행
    
    ```bash
    # Pod 생성
    kubectl apply -f counter-pod-redis.yml
    
    # Pod 목록 조회
    kubectl get pod
    
    # Pod 로그 확인
    kubectl logs counter # 오류 발생 (컨테이너 지정 필요)
    kubectl logs counter app
    kubectl logs counter db
    
    # Pod의 app 컨테이너 접속
    kubectl exec -it counter -c app -- sh
    # apk add curl busybox-extras # install curl, telnet
    # curl localhost:3000
    # curl localhost:3000
    # telnet localhost 6379
      dbsize
      KEYS *
      GET count
      quit
    
    # Pod 제거
    kubectl delete -f counter-pod-redis.yml
    ```
    

# 🌱  ReplicaSet

### **ReplicaSet이란?**

Replicaset은 정해진 수의 복제된 pod가 반드시 실행되고 있음을 보장해준다. 

Pod를 하나만 만들면, Pod에 문제가 생겼을 때, 자동으로 복구되지 않는다.

ReplicaSet은 주어진 시간동안 Pod를 정해진 수만큼 복제하고 관리하는 것을 담당한다.

### **ReplicaSet 만들기**

- ReplicaSet 요소
    - `spec.selector` : label 체크 조건
    - `spec.replicas` : 동시에 실행되어야할 Pod의 개수.
    - `spec.template` : 파드의 template. 생성할 Pod의 명세
- echo-rs.yml
    
    ```yaml
    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: echo-rs
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: echo
          tier: app
      template:
        metadata:
          labels:
            app: echo
            tier: app
        spec:
          containers:
            - name: echo
              image: ghcr.io/subicura/echo:v1
    ```
    
- 실행 및 결과 확인
    
    ```bash
    # ReplicaSet 생성
    > k apply -f echo-rs.yml
    replicaset.apps/echo-rs created
    
    # 리소스 확인
    > k get pod,rs
    NAME                READY   STATUS    RESTARTS   AGE
    pod/echo-rs-x6gwr   1/1     Running   0          76s
    
    NAME                      DESIRED   CURRENT   READY   AGE
    replicaset.apps/echo-rs   1         1         1       76s
    ```
    

ReplicaSet은 

1. `selector` 에서 명시한 label을 체크해서
2. 원하는 수(`replicas`)의 Pod가 없으면
3. `template` 에 정의한 명세를 따르는 새로운 Pod를 생성한다.

- 파드의 라벨 확인
    
    ```bash
    # 파드의 라벨 확인
    > k get pod --show-labels
    NAME            READY   STATUS    RESTARTS   AGE     LABELS
    echo-rs-x6gwr   1/1     Running   0          8m24s   app=echo,tier=app
    ```
    
- 파드의 라벨 제거
    
    ```bash
    # app-로 파드 pod/echo-rs-x6gw의 app 라벨 제거
    > k label pod/echo-rs-x6gwr app-
    pod/echo-rs-x6gwr labeled
    
    # 라벨 제거 결과 확인
    > k get pod --show-labels
    NAME            READY   STATUS    RESTARTS   AGE     LABELS
    echo-rs-g8hlw   1/1     Running   0          17s     app=echo,tier=app
    echo-rs-x6gwr   1/1     Running   0          9m55s   tier=app
    ```
    
    app 라벨을 제거함으로써, selector에서 정의한 `app=echo,tier=app` 조건을 만족하는 파드의 개수가 0이 되었으므로, template에 맞춰 새로운 파드가 생성되었다.
    
- 다시 app 라벨 추가
    
    ```bash
    # app=echo로 app 라벨 추가
    > k label pod/echo-rs-x6gwr app=echo
    pod/echo-rs-x6gwr labeled
    
    # 라벨 추가 결과 확인
    k get pod --show-labels
    NAME            READY   STATUS        RESTARTS   AGE   LABELS
    echo-rs-g8hlw   0/1     Terminating   0          56s   app=echo,tier=app
    echo-rs-x6gwr   1/1     Running       0          10m   app=echo,tier=app
    ```
    
    replicas에는 파드의 수를 1개로 유지하기로 하였으므로, 기존 Pod는 제거한다. 
    
    echo-rs-g8hlw는 Terminating 상태에 있고, 곧 삭제된다. 
    

### **ReplicaSet 동작 방식**

![스크린샷 2021-10-27 오전 5.08.33.png](/public/2021-10-27_5.08.33.png)

1. ReplicaSet Controller가 ReplicaSet 조건을 감지하면서, 현재 상태와 원하는 상태가 다른지 확인한다.
2. ReplicaSet Controller가 원하는 상태가 되도록 Pod를 생성하거나 제거한다.
3. Scheduler는 API서버를 감시하면서, 노드에 할당되지 않은 새롭게 생성된 Pod가 있는지 확인한다.
4. Scheduler는 할당되지 않은 새로운 Pod를 감지하고, 적절한 노드에 배치한다.

# 🌱  Deployment

![스크린샷 2021-10-24 오후 4.54.04.png](/public/2021-10-24_4.54.04.png)

### **deployment란?**

- Deployment는 ReplicaSet의 상위 레벨의 개념이다.
- Deployment는 ReplicaSet을 이용하여 새로운 ReplicaSet을 만들 수 있고, 버전을 관리하여 롤백하거나 특정 버전으로 되돌아갈 수 있다.
- Deployment에 원하는 상태(desired state)를 적고, Deployment Controller가 실제 상태를 원하는 상태로 변경시킨다.

### d**eployment 만들기**

- echo-dp.yml
    - `spec.replicas` : 복제할 파드의 수.
    - `spec.selector` : Deployment가 어떤 Pod를 관리할 것인지를 정의. 여기서는 pod의 라벨을 기술.
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: echo-deploy
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: echo
          tier: app
      template:
        metadata:
          labels:
            app: echo
            tier: app
        spec:
          containers:
            - name: echo
              image: ghcr.io/subicura/echo:v1
    ```
    
- 실행 및 결과 확인
    - `READY` : 몇 개의 Replica를 사용가 사용할 수 있는지
    - `UP-TO-DATE` : 몇 개의 Replica가 원하는 상태가 되기 위해 업데이트 되었는지
    - `AVAILABLE` : 몇 개의 Replica를 사용자가 사용할 수 있는지
    - `AGE` : 애플리케이션이 실행된 시간
    
    ```bash
    # Deployment 생성
    > k apply -f echo-dp.yml
    deployment.apps/echo-deploy created
    
    # 리소스 확인
    > k get pod,rs,deploy
    NAME                               READY   STATUS    RESTARTS   AGE
    pod/echo-deploy-76dcd9f4f9-2m7jb   1/1     Running   0          63s
    pod/echo-deploy-76dcd9f4f9-459h4   1/1     Running   0          63s
    pod/echo-deploy-76dcd9f4f9-g2md8   1/1     Running   0          63s
    pod/echo-deploy-76dcd9f4f9-lttbg   1/1     Running   0          63s
    
    NAME                                     DESIRED   CURRENT   READY   AGE
    replicaset.apps/echo-deploy-76dcd9f4f9   4         4         4       63s
    
    NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/echo-deploy   4/4     4            4           63s
    ```
    

### **Pod를 새로운 이미지로 업데이트하기**

deployment의 template이 변경되면, rollout이 실행

- echo-dp.yml
    
    ```bash
    # 새로운 이미지 업데이트
    > kubectl set image deployment.apps/echo-deploy echo=ghcr.io/subicura/echo:v2
    deployment.apps/echo-deploy image updated
    ```
    
- 실행 및 결과 확인
    
    새로운 버전의 pod가 생성되고, 기존 pod는 제거된다.
    
    기존의 ReplicaSet은 0으로 scale down되었고, 새로운 ReplicaSet이 생성되엇으며 4개로 scale up된 것을 확인할 수 있다.
    
    ```bash
    # 리소스 확인
    > kubectl get po,rs,deploy
    NAME                               READY   STATUS        RESTARTS   AGE
    pod/echo-deploy-76dcd9f4f9-2m7jb   0/1     Terminating   0          6m39s
    pod/echo-deploy-76dcd9f4f9-459h4   0/1     Terminating   0          6m39s
    pod/echo-deploy-76dcd9f4f9-g2md8   0/1     Terminating   0          6m39s
    pod/echo-deploy-76dcd9f4f9-lttbg   0/1     Terminating   0          6m39s
    pod/echo-deploy-77cd7699f4-4kbtj   1/1     Running       0          4s
    pod/echo-deploy-77cd7699f4-plrjb   1/1     Running       0          3s
    pod/echo-deploy-77cd7699f4-qz2hn   1/1     Running       0          10s
    pod/echo-deploy-77cd7699f4-t98f5   1/1     Running       0          10s
    
    NAME                                     DESIRED   CURRENT   READY   AGE
    replicaset.apps/echo-deploy-76dcd9f4f9   0         0         0       6m39s
    replicaset.apps/echo-deploy-77cd7699f4   4         4         4       10s
    
    NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/echo-deploy   4/4     4            4           6m39s
    
    # replicaset 확인
    > k get replicaset
    NAME                     DESIRED   CURRENT   READY   AGE
    echo-deploy-76dcd9f4f9   0         0         0       8m50s
    echo-deploy-77cd7699f4   4         4         4       2m45s
    ```
    

deployment는 이미지를 업데이트하기 위해 ReplicaSet을 이용한다. 

버전을 업데이트하면, 새로운 ReplicaSet이 생성되고, 해당 ReplicaSet이 새로운 버전의 Pod를 생성한다.

![스크린샷 2021-10-27 오전 5.43.33.png](/public/2021-10-27_5.43.33.png)

... 이 과정을 쭉 거치면, 최종적으로 새로운 Replicaset에 이미지가 v2인 Pod가 4개 생성된다.

![스크린샷 2021-10-27 오전 5.44.02.png](/public/2021-10-27_5.44.02.png)

생성한 Deployment의 상세 상태에서 Event를 보면, 그 과정을 확인할 수 있다.

```bash
# 상세 상태 확인
> k describe deploy/echo-deploy
Name:                   echo-deploy
Namespace:              default
...(생략)...
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  9m10s  deployment-controller  Scaled up replica set echo-deploy-76dcd9f4f9 to 4
  Normal  ScalingReplicaSet  2m41s  deployment-controller  Scaled up replica set echo-deploy-77cd7699f4 to 1
  Normal  ScalingReplicaSet  2m41s  deployment-controller  Scaled down replica set echo-deploy-76dcd9f4f9 to 3
  Normal  ScalingReplicaSet  2m41s  deployment-controller  Scaled up replica set echo-deploy-77cd7699f4 to 2
  Normal  ScalingReplicaSet  2m35s  deployment-controller  Scaled down replica set echo-deploy-76dcd9f4f9 to 2
  Normal  ScalingReplicaSet  2m35s  deployment-controller  Scaled up replica set echo-deploy-77cd7699f4 to 3
  Normal  ScalingReplicaSet  2m34s  deployment-controller  Scaled down replica set echo-deploy-76dcd9f4f9 to 1
  Normal  ScalingReplicaSet  2m34s  deployment-controller  Scaled up replica set echo-deploy-77cd7699f4 to 4
  Normal  ScalingReplicaSet  2m34s  deployment-controller  Scaled down replica set echo-deploy-76dcd9f4f9 to 0
```

### **Deployment 동작 방식**

![스크린샷 2021-10-27 오전 5.50.10.png](/public/2021-10-27_5.50.10.png)

1. `Deployment Controller`는 Deployment 조건을 감시하면서 현재 상태와 원하는 상태가 다른 것을 체크
2. `Deployment Controller`가 원하는 상태가 되도록 `ReplicaSet` 설정
3. `ReplicaSet Controller`는 ReplicaSet조건을 감시하면서 현재 상태와 원하는 상태가 다른 것을 체크
4. `ReplicaSet Controller`가 원하는 상태가 되도록 `Pod`을 생성하거나 제거
5. `Scheduler`는 API서버를 감시하면서 할당되지 않은 `Pod`이 있는지 체크
6. `Scheduler`는 할당되지 않은 새로운 `Pod`을 감지하고 적절한 `노드`에 배치
7. 이후 노드는 기존대로 동작

### **버전관리**

- rollout 히스토리 확인
    
    ```bash
    # 히스토리 확인
    > kubectl rollout history deploy/echo-deploy
    deployment.apps/echo-deploy
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    
    # revision 1 히스토리 상세 확인
    > kubectl rollout history deploy/echo-deploy --revision=1
    deployment.apps/echo-deploy with revision #1
    Pod Template:
      Labels:	app=echo
    	pod-template-hash=76dcd9f4f9
    	tier=app
      Containers:
       echo:
        Image:	ghcr.io/subicura/echo:v1
        Port:	<none>
        Host Port:	<none>
        Environment:	<none>
        Mounts:	<none>
      Volumes:	<none>
    ```
    
- 롤백
    
    ```yaml
    # 바로 전으로 롤백
    kubectl rollout undo deploy/echo-deploy
    
    # 특정 버전으로 롤백
    kubectl rollout undo deploy/echo-deploy --to-revision=2
    ```
    

### 배포 전략 설정

Deployment에서 롤링업데이트( RollingUpate ) 배포 전략을 사용하여, 동시에 업데이트하는 Pod의 갯수를 변경시킬 수 있다. 

- echo-strategy.yml
    - `.spec.strategy` : 새로운 파드로 교체하는 전략
    - `.spec.strategy.type`
        - Recreate : 새로운 파드가 생성되기 전에 기존 파드를 전부 죽인다.
        - RollingUpdate (default) : maxUnavailable과 maxSurge를 지정할 수 있다.
    - `.spec.strategy.rollingUpdate.maxSurge` : desired pod 갯수 이상으로 생성될 수 있는 파드의 최대 수 or 비율이다. 예를 들어 30%면, desired pod의 수는 130%를 넘을 수 없다. 디폴트는 25%이다.
    - `.spec.strategy.rollingUpdate.maxUnavailable` : 업데이트하는 중에 unavailable한 파드의 최대 수 or 비율이다. 디폴트는 25%이다.
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: echo-strategy
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: echo
          tier: app
      minReadySeconds: 5
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxSurge: 3
          maxUnavailable: 3
      template:
        metadata:
          labels:
            app: echo
            tier: app
        spec:
          containers:
            - name: echo
              image: ghcr.io/subicura/echo:v1
              livenessProbe:
                httpGet:
                  path: /
                  port: 3000
    ```
    
- 실행 및 결과
    
    이미지를 변경하면, 한 번에 3개씩 변경이 되는 것을 확인할 수 있다. 
    
    ```bash
    # 실행
    > k apply -f echo-strategy.yml 
    deployment.apps/echo-strategy created
    
    # 이미지 변경 
    > kubectl set image deploy/echo-strategy echo=ghcr.io/subicura/echo:v2
    
    # 이벤트 확인
    > k describe deployments.apps
    ...
    Events:
      Type    Reason             Age   From                   Message
      ----    ------             ----  ----                   -------
      Normal  ScalingReplicaSet  40s   deployment-controller  Scaled up replica set echo-strategy-65cd57dc46 to 4
      Normal  ScalingReplicaSet  18s   deployment-controller  Scaled up replica set echo-strategy-5874d6bb79 to 3
      Normal  ScalingReplicaSet  18s   deployment-controller  Scaled down replica set echo-strategy-65cd57dc46 to 1
      Normal  ScalingReplicaSet  18s   deployment-controller  Scaled up replica set echo-strategy-5874d6bb79 to 4
      Normal  ScalingReplicaSet  10s   deployment-controller  Scaled down replica set echo-strategy-65cd57dc46 to 0
    ```
    

# 🌱  Service

![스크린샷 2021-10-27 오전 8.33.49.png](/public/2021-10-27_8.33.49.png)

### **Service란?**

Service는 네트워크 서비스로, set of pods에 접근하는 추상적인 방법을 제공한다. 

각 Pod는 고유한 IP 주소를 갖고 있고, 같은 라벨인 Pod끼리는 하나의 DNS 이름을 갖고 있다. 

문제는 이 Pod들이 죽고 새로 생성될 때마다 새로운 IP주소를 갖게 되는 것에 있다. 

이 Pod들에 접근해야하는 다른 Pod들은 IP주소가 변경되는 것을 감지하고, 변경된 IP주소에 맞춰줘야하는 문제가 있다. 

이 문제를 해결한 것이 Service이다. Service는 가상 IP를 갖고 있으며, 이 주소는 Pod가 죽더라도 변경되지 않는다.

따라서 set of pod에 접근하려면, Service로 접근하면 된다. 

서비스는 노출 범위에 따라 `ClusterIP` , `NodePort`, `LoadBalancer`  타입으로 나뉜다.

- `ClusterIP` : 서비스를 cluster-internal IP로 노출한다. 서비스는 오직 클러스터 내부에서만 접근이 가능하다.
- `NodePort` : 서비스를 각 노드의 IP에서 고정된 port로 노출한다. <NodeIP>:<NodePort>로 요청하면, 클러스터 외부에서 서비스에 접근할 수 있다.
- `LoadBalancer` : 서비스를 클러스터 프로바이더의 로드밸런서를 사용ㅎ

### **Service - ClusterIP 생성하기**

ClusterIP는 클러스터 내부에 새로운 IP를 할당하고, 여러 개의 Pod를 바라보는 로드밸런서의 기능을 제공한다. 

서비스 이름을 내부 도메인 서버에 등록하여 파드 간에 서비스 이름을 통신할 수 있다. 

- counter-redis-svc.yml
    - redis를 서비스로 노출
    - YAML 파일에서 여러 개의 소스를 정의할 땐 `---` 구분자 사용
    - 같은 클러스터에서 생성된 Pod는 `redis` 라는 이름으로 redis Pod에 접근 가능
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: redis
    spec:
      selector:
        matchLabels:
          app: counter
          tier: db
      template:
        metadata:
          labels:
            app: counter
            tier: db
        spec:
          containers:
            - name: redis
              image: redis
              ports:
                - containerPort: 6379
                  protocol: TCP
    
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: redis
    spec:
      ports:
        - port: 6379
          protocol: TCP
      selector:
        app: counter
        tier: db
    ```
    

- ClusterIP 설정
    - `spec.ports.port` : 서비스가 생성할 Port. 서비스에 들어오는 port를 targetPort로 매칭시킨다.
    - `spec.ports.targetPort` : 서비스가 접근할 Pod의 Port. 편의성을 위해 targetPort는 port와 같은 값이 디폴트이다.
    - `spec.selector` : Service가 접근할 Pod들의 라벨 조건
- 실행 및 결과 확인
    
    ```bash
    > k apply -f counter-redis-svc.yml
    deployment.apps/redis created
    service/redis created
    
    # pod, replicaSet, Deployment, Service 리소스 확인
    > k get all
    NAME                         READY   STATUS              RESTARTS   AGE
    pod/redis-57d787df44-gn4zc   0/1     ContainerCreating   0          4s
    
    NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    6h45m
    service/redis        ClusterIP   10.109.151.206   <none>        6379/TCP   4s
    
    NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/redis   0/1     1            0           4s
    
    NAME                               DESIRED   CURRENT   READY   AGE
    replicaset.apps/redis-57d787df44   1         1         0       4s
    ```
    
- counter app을 deployment로 만들기
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: counter
    spec:
      selector:
        matchLabels:
          app: counter
          tier: app
      template:
        metadata:
          labels:
            app: counter
            tier: app
        spec:
          containers:
            - name: counter
              image: ghcr.io/subicura/counter:latest
              env:
                - name: REDIS_HOST
                  value: "redis"
                - name: REDIS_PORT
                  value: "6379"
    ```
    
- counter app Pod에서 redis Pod로 접근이 되는지 테스트
    
    같은 클러스터에서 생성된 Pod는 `redis` 라는 도메인으로 redis Pod에 접근 가능하다.
    
    Service를 통해 Pod에 연결이 된 것을 확인할 수 있다. 
    
    ```bash
    > kubectl apply -f counter-app.yml
    
    # counter app에 접근
    > kubectl get pod
    NAME                       READY   STATUS    RESTARTS   AGE
    counter-7b76fd9bc6-xp6bv   1/1     Running   0          21s
    redis-57d787df44-gn4zc     1/1     Running   0          10m
    
    > kubectl exec -it counter-7b76fd9bc6-xp6bv -- sh
    # apk add curl busybox-extras # install telnet
    # curl localhost:3000
    # curl localhost:3000
    # telnet redis 6379
      dbsize
      KEYS *
      GET count
      quit
    ```
    

### **Service 생성 흐름**

![스크린샷 2021-10-27 오전 9.16.59.png](/public/2021-10-27_9.16.59.png)

Service는 각 Pod를 바라보는 로드밸런서 역할을 하면서, 내부 도메인서버에 새로운 도메인을 생성한다.

1. `Endpoint Controller`는 `Service`와 `Pod`을 감시하면서 조건에 맞는 **Pod의 IP를 수집**
2. `Endpoint Controller`가 수집한 IP를 가지고 `Endpoint` 생성
3. `Kube-Proxy`는 `Endpoint` 변화를 감시하고 노드의 iptables을 설정
    - kube-proxy는 내부에서 iptables를 사용해서 virtual IP 주소를 정의한다.
    - 클라이언트가 서비스의 virtual IP로 연결하면, 서비스로 들어온 요청을 Pod에 전달하는 역할을 한다.
    - `Kube-Proxy` 는 각 노드에서 실행되는 네트워크 프록시이다.
4. `CoreDNS`는 `Service`를 감시하고 서비스 이름과 IP를 `CoreDNS`에 추가
    - `CoreDNS` 는 클러스터 내부용 DNS이다. CoreDNS를 사용하여, IP 대신 도메인 이름을 사용한다.

> iptables
> 

iptables는 규칙이 많아지면, 성능이 느려지는 이슈가 있어서 ipvs를 사용하는 옵션도 있다.

> Endpoint
> 

`Endpoint` 는 서비스의 접속 정보를 갖고 있다.

```bash
# endpoint 확인하기
> k get endpoints
NAME         ENDPOINTS           AGE
kubernetes   192.168.49.2:8443   7h7m
redis        172.17.0.3:6379     22m

# redis endpoint의 정보
> k describe ep/redis
Name:         redis
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2021-10-26T23:58:22Z
Subsets:
  Addresses:          172.17.0.3
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  6379  TCP

# redis pod의 IP 정보
> k describe pod/redis-57d787df44-gn4zc
Name:         redis-57d787df44-gn4zc
...(생략)...
IP:           172.17.0.3
```

### Service - NodePort 만들기

ClusterIP는 클러스터 내부에서만 접근할 수 있지만, NodePort는 클러스터 외부에서도 접근할 수 있다. 

- counter-nodeport.yml
    - `spec.ports.nodeport` : 노드에 오픈할 port. 미지정하면 30000-32768 중에 자동 할당.
    
    ```bash
    apiVersion: v1
    kind: Service
    metadata:
      name: counter-np
    spec:
      type: NodePort
      ports:
        - port: 3000
          protocol: TCP
          nodePort: 31000
      selector:
        app: counter
        tier: app
    ```
    

### Service - NodePort

NodePort는 서비스를 클러스터 외부에서도 접근할 수 있도록 해준다. 

![스크린샷 2021-10-30 오후 7.57.07.png](/public/2021-10-30_7.57.07.png)

외부의 IP인 192.168.1.1의 31000포트를 열고, 노드에 접근할 수 있게 한다. 

192.168.1.1:31000으로 서비스를 외부로 노출시키고, spec.selecor로 지정한 pod들이 서비스하게 된다. 

- nginx-deployment.yml
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      labels:
        app: nginx
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.14.2
            ports:
            - containerPort: 80
    ```
    
- nginx-service.yml
    - `spec.ports.nodePort` : 서비스를 노출할 포트 지정
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-nodeport-service
      labels:
        run: nginx-nodeport-service
    spec:
      type: NodePort     # 서비스 타입
      ports:
      - port: 80       # 서비스 포트
        targetPort: 80   # 타켓, 즉 pod의 포트
        protocol: TCP
        nodePort: 31000
        name: http
      selector:
        app: nginx
    ```
    
- 실행
    
    ```yaml
    # deployment와 service 실행 
    > k apply -f nginx-deployment.yml
    > k apply -f nginx-service.yml
    
    # 상태 확인 
    
    # minikube service <service-name> 특정 서비스를 브라우저에 오픈
    > minikube service nginx-nodeport-service
    |-----------|------------------------|-------------|---------------------------|
    | NAMESPACE |          NAME          | TARGET PORT |            URL            |
    |-----------|------------------------|-------------|---------------------------|
    | default   | nginx-nodeport-service | http/80     | http://192.168.49.2:31000 |
    |-----------|------------------------|-------------|---------------------------|
    🏃  nginx-nodeport-service 서비스의 터널을 시작하는 중
    |-----------|------------------------|-------------|------------------------|
    | NAMESPACE |          NAME          | TARGET PORT |          URL           |
    |-----------|------------------------|-------------|------------------------|
    | default   | nginx-nodeport-service |             | http://127.0.0.1:52732 |
    |-----------|------------------------|-------------|------------------------|
    🎉  Opening service default/nginx-nodeport-service in default browser...
    ```
    
    <NodeIP>:<NodePort> (http://192.168.49.2:31000) 로 브라우저에 접근하면, Nginx 화면이 보이게 된다.
    

# 🌱 Ingress

![스크린샷 2021-10-27 오전 9.51.18.png](/public/2021-10-27_9.51.18.png)

### **Ingress 란?**

- 클러스터 외부의 HTTP와 HTTPS 라우트를 클러스터 내의 서비스에 연결시킨다.
- 공인IP로 클러스터 외부에서 서비스에 접근할 수 있게 해준다.
- ingress가 동작하려면, ingress controller가 필요하다. ingress controller는 별도 설치가 필요하다.

이전에 보았듯이 NodePort로도 서비스를 외부에 노출시킬 수 있지만, NodePort는 서비스의 갯수만큼 포트를 오픈시켜야 한다. Ingress를 이용하면, 80이나 443 포트로 여러 개의 서비스를 연결시킬 수 있다. 

### **Ingress 만들기**

echo 웹앱을 버전별(v1, v2)로 도메인을 다르게 만들기

1. minikube에 ingress 활성화하기
    
    ingress는 별도의 컨트롤러를 설치해야 한다.
    
    여기서는 nginx ingress controller를 사용한다.
    
    ```bash
    # MacOS의 docker drive에서 ingress addon이 안되는 이슈가 있어, vm-based driver를 시작한다.
    > minikube delete
    > minikube start --vm=true
    
    # ingress addon 사용
    > minikube addons enable ingress
    
    # nginx ingress ingress 컨트롤러가 실행되는지 확인
    > kubectl get pods -n ingress-nginx
    NAME                                        READY   STATUS      RESTARTS   AGE
    ingress-nginx-admission-create-5bqzm        0/1     Completed   0          3m
    ingress-nginx-admission-patch-xb4zm         0/1     Completed   0          3m
    ingress-nginx-controller-5d88495688-66xbz   1/1     Running     0          3m
    ```
    

1. echo 웹앱 배포
- echo-v1.yml
    - `kind` : Ingress
    - `Spec`
        - 로드 밸런서나 프록시 서버를 설정하는데 필요한 정보를 담고 있다.
        - 모든 요청에 대응하는 rule들을 작성한다.
        - Ingress는 HTTP, HTTPS 트래픽에 대한 rule만을 지원한다.
    - Ingress Rule
        - `host` : 이 룰이 적용될 host. 만약 host를 지정하지 않으면 IP주소를 통해 들어오는 HTTP 트래픽이 적용된다.
        - `path` :  연관된 backend에 대응된다. backend에는 service의 name과 port가 지정된다.
        - `pathType`: path가 어떻게 매칭이 되도록 할 것인지에 대한 타입을 지정한다.
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: echo-v1
    spec:
      rules:
        - host: v1.echo.192.168.64.2.sslip.io
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: echo-v1
                    port:
                      number: 3000
    
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: echo-v1
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: echo
          tier: app
          version: v1
      template:
        metadata:
          labels:
            app: echo
            tier: app
            version: v1
        spec:
          containers:
            - name: echo
              image: ghcr.io/subicura/echo:v1
              livenessProbe:
                httpGet:
                  path: /
                  port: 3000
    
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: echo-v1
    spec:
      ports:
        - port: 3000
          protocol: TCP
      selector:
        app: echo
        tier: app
        version: v1
    ```
    
- echo-v2.yml
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: echo-v2
    spec:
      rules:
        - host: v2.echo.192.168.64.2.sslip.io
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: echo-v2
                    port:
                      number: 3000
    
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: echo-v2
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: echo
          tier: app
          version: v2
      template:
        metadata:
          labels:
            app: echo
            tier: app
            version: v2
        spec:
          containers:
            - name: echo
              image: ghcr.io/subicura/echo:v2
              livenessProbe:
                httpGet:
                  path: /
                  port: 3000
    
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: echo-v2
    spec:
      ports:
        - port: 3000
          protocol: TCP
      selector:
        app: echo
        tier: app
        version: v2
    ```
    
1. 실행 및 결과
    
    ```bash
    > k apply -f echo-v1.yml
    > k apply -f echo-v2.yml
    
    # ingress 상태 확인
    > k get ingress
    NAME      CLASS    HOSTS                           ADDRESS   PORTS   AGE
    echo-v1   <none>   v1.echo.192.168.64.2.sslip.io             80      17s
    echo-v2   <none>   v2.echo.192.168.64.2.sslip.io             80      22s
    
    # ingress/echo-v1 상세 정보 확인
    > k describe ingress/echo-v1
    Name:             echo-v1
    Namespace:        default
    Address:          192.168.64.2
    Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
    Rules:
      Host                           Path  Backends
      ----                           ----  --------
      v1.echo.192.168.64.2.sslip.io
                                     /   echo-v1:3000 (172.17.0.7:3000,172.17.0.8:3000,172.17.0.9:3000)
    Annotations:                     <none>
    Events:
      Type    Reason  Age                    From                      Message
      ----    ------  ----                   ----                      -------
      Normal  Sync    4m22s (x2 over 4m58s)  nginx-ingress-controller  Scheduled for sync
    ```
    
    v1.echo.192.168.64.2.sslip.io와 v2.echo.192.168.64.2.sslip.io로 접속 테스트한다.
    

### **Ingress 생성 흐름**

![스크린샷 2021-10-27 오전 10.49.50.png](/public/2021-10-27_10.49.50.png)

1. `Ingress Controller`는 `Ingress` 변화를 체크
2. `Ingress Controller`는 변경된 내용을 `Nginx`에 설정하고 프로세스 재시작

ingress를 사용하면, YAML 설정만 변경하면 도메인과 경로 설정을 쉽게 바꿀 수 있다. 

# Reference

[기본 명령어](https://subicura.com/k8s/guide/kubectl.html#%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%BC-%E1%84%80%E1%85%AA%E1%86%AB%E1%84%85%E1%85%B5-config)

[kubectl 개요](https://kubernetes.io/ko/docs/reference/kubectl/overview/)

[Pods](https://kubernetes.io/docs/concepts/workloads/pods/)

[ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

[docker: Ingress not exposed on MacOS · Issue #7332 · kubernetes/minikube](https://github.com/kubernetes/minikube/issues/7332)

[Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

[Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
