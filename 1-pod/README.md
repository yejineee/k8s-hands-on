# Pod

> **Pod 란?**
> 
- 쿠버네티스에서 관리하는 가장 작은 배포 단위
- Pod는 하나 이상의 컨테이너로 구성되며, **컨테이너끼리는 스토리지와 네트워크 자원을 공유**한다.
- In terms of Docker concepts, a Pod is similar to a group of Docker containers with **shared namespaces and shared filesystem volumes.**

> **Pod 생성하기**
> 

클러스터에서 특정 이미지를 실행시킨다.

- command : `kubectl run`
    
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
    
    - `kubectl describe`  : 리소스의 상세한 상태 정보를 확인할 수 있다.
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
    

> **Pod 제거하기** - `kubectl delete`
> 

```bash
> kubectl delete pod/echo
pod "echo" deleted
```

> **YAML파일에 오브젝트 스펙 작성하기**
> 

실전에서는 `kubectl run` 명령어로 파드를 직접 생성하기 보다는, YAML 파일에 원하는 리소스를 작성하여 관리한다. 

- kubectl run

```bash
kubectl run echo --image ghcr.io/subicura/echo:v1
```

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

[필수요소](https://www.notion.so/984b487cdbae4651a97dd416a1190ad9)

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
    

> **Pod 생성 분석하기**
> 

![스크린샷 2021-10-27 오전 3.38.34.png](./images/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2021-10-27_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_3.38.34.png)

API Server는 `kubectl run` 명령어로 새로운 Pod를 생성하라는 요청을 받는다.

1. Scheduler는 노드가 할당되지 않은 새롭게 생성된 Pod가 있는지 확인한다.
2. Scheduler는 노드가 할당되지 않은 파드를 감지하고, 해당 파드를 실행시킬 적절한 노드를 선택한다.
3. kubelet은 자신의 노드에 할당된 Pod가 있는지 확인한다.
4. kubelet은 Scheduler에 의해 자신에게 할당된 Pod의 정보를 확인하고 컨테이너를 생성한다.
5. kubelet은 자신에게 할당된 Pod의 상태를 API Server에 전달한다.

> **컨테이너 상태 모니터링**
> 

![스크린샷 2021-10-30 오후 3.52.02.png](./images/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2021-10-30_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.52.02.png)

- `컨테이너 생성` 과 실제 `서비스 준비` 는 차이가 있다. 쿠버네티스는 컨테이너가 생성되고, 서비스가 준비되었다는 것을 체크하는 옵션을 제공하여 초기화하는 동안 서비스되는 것을 막을 수 있다.
- kubelet은 주기적으로 컨테이너의 상태를 진단한다. 이를 Probe라고 한다.
- kubelet은 컨테이너 내부에 있는 핸들러를 호출하여, 컨테이너 상태를 진단한다.
    - 핸들러에는 세 가지 타입이 있다. →  `httpGet`, `tcpSocket`, `exec`
    - 진단의 결과는 세 가지 중 하나이다. → `Success`, `Failure`, `Unknown` (진단 자체가 실패)
    - 세 가징 종류의 probe를 할 수 있다. → `livenessProbe`, `readinessProbe`, `startupProbe`

> **livenessProbe**
> 
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
    

> **readinessProbe**
> 
- 컨테이너가 요청에 응답할 수 있는 상태인지를 체크한다.
- readinessProbe가 실패하면, Endpoint Controller는 Pod의 IP를 모든 서비스의 endpoint에서 지운다. → Pod로 요청이 들어오지 못하게 한다.
- livenessProbe와의 차이점은 probe가 실패하여도, Pod를 재시작하지 않고 요청만 제외한다는 것이다.
- 언제 readiness probe를 사용해야 하는가?
    
    probe가 성공적일 경우에만, 파드로 트래픽을 보내고 싶을 때 readinessProbe를 명시해야 한다. 
    
- echo-rp.yml
    
    ```bash
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
    

> livenessProbe + readinessProbe
> 
- 애플리케이션이 백엔드 서비스와 강하게 연관이 있을 때는 liveness와 readiness probe 둘 다 할 수 있다. liveness는 앱 자체가 healthy하면 통과하지만, readiness는 앱에서 필요한 각 백엔드 서비스가 모두 사용가능한지를 체크한다.

> **다중 컨테이너**
> 

파드는 하나 이상의 컨테이너를 갖고 있지만, 대부분 하나의 파드에 하나의 컨테이너가 있다. (one-container-per-Pod 모델)

리소스를 공유해야하는 경우엔, 한 파드에 여러 컨테이너를 갖고 있을 수도 있다. 

한 파드에 속한 컨테이너들은 (1) **서로 네트워크를 localhost로 공유**하고, **(2) 같은 디렉토리를 공유**할 수 있다.

- 요청 횟수를 redis에 저장하는 웹앱
    - 한 파드 안에 counter앱과 redis 컨테이너가 있으므로, counter 앱은 redis를 localhost로 접근할 수 있다.
        
        ![스크린샷 2021-10-27 오전 8.48.05.png](./images/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2021-10-27_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_8.48.05.png)
        
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
