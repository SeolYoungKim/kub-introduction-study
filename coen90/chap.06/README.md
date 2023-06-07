# 6장 컨트롤러

## 6.1 레플리케이션 컨트롤러

- 지정한 숫자만큼 파드가 항상 클러스터 안에서 실행되도록 관리
- 컨트롤러를 사용하지 않고 파드를 직접 실행하면 파드에 이상이 생겨, 종료되거나 삭제했을 때 재시작하기 어렵다
- 요즘은 레플리카세트를 사용하는 추세이며, 앱 배포에는 디플로이먼트를 주로 사용한다.

## 6.2 레플리카 세트

- 레플리케이션 컨트롤러의 발전형
- in, notin, exists 와 같은 집합 기반의 셀렉터를 지원
- rolling-update 옵션 사용 가능

### 6.2.1 레플리카세트 사용

```yaml
# replicaset/replicaset-nginx.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  template: # 레플리카세트가 어떤 파드를 실행할지에 관한 정보 설정
    metadata:
      name: nginx-replicaset
      labels:
        app: nginx-replicaset
    spec:
      containers: # 컨테이너의 구체적인 명세 설정
      - name: nginx-replicaset
        image: nginx
        ports:
        - containerPort: 80
  replicas: 3 # 파드를 몇 개 유지할지 개수
  selector: # 어떤 레이블의 파드를 선택해 관리할지
    matchLabels: # 위 .spec.template.metadata.labels의 설정과 같아야 함
      app: nginx-replicaset
```

```bash
$ kubectl apply -f replicaset-nginx.yaml
replicaset.apps/nginx-replicaset created

# 파드가 3개 생성됨
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-mpwkt   1/1     Running   0          37s
nginx-replicaset-vn7qk   1/1     Running   0          37s
nginx-replicaset-zzctg   1/1     Running   0          37s

# 하나 지워보기
$ kubectl delete pod nginx-replicaset-mpwkt
pod "nginx-replicaset-mpwkt" deleted

# 또 생성됨
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-rb2d9   1/1     Running   0          2s
nginx-replicaset-vn7qk   1/1     Running   0          79s
nginx-replicaset-zzctg   1/1     Running   0          79s
```

### 6.2.2 레플리카세트와 파드의 연관관계

```bash
$ kubectl get replicaset,pods
NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-replicaset   3         3         3       3m32s
# DESIRED: 레플리카세트에 지정한 파드 개수 CURRENT 실제 파드 개수

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-replicaset-rb2d9   1/1     Running   0          2m14s
pod/nginx-replicaset-vn7qk   1/1     Running   0          3m31s
pod/nginx-replicaset-zzctg   1/1     Running   0          3m31s

$ kubectl delete replicaset nginx-replicaset --cascade=false # --cascase 옵션 삭제하면 파드까지 삭제됨
warning: --cascade=false is deprecated (boolean value) and can be replaced with --cascade=orphan.
replicaset.apps "nginx-replicaset" deleted

$ kubectl get replicaset,pods
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-replicaset-rb2d9   1/1     Running   0          2m39s
pod/nginx-replicaset-vn7qk   1/1     Running   0          3m56s
pod/nginx-replicaset-zzctg   1/1     Running   0          3m56s
```

- [metadata.labels.app](http://metadata.labels.app) 을 바꾸면 어떻게 될까 (nginx-replicaset → nginx-others)

```bash
$ kubectl edit pod/nginx-replicaset-6mk6l
pod/nginx-replicaset-6mk6l edited
```

```bash
$ kubectl get pods -o=jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.metadata.labels}{'\n'}{end}"
nginx-replicaset-6mk6l  {"app":"nginx-others"}
nginx-replicaset-9z79s  {"app":"nginx-replicaset"}
nginx-replicaset-m9959  {"app":"nginx-replicaset"}
nginx-replicaset-nnt7z  {"app":"nginx-replicaset"}
```

## 6.3 디플로이먼트

- 상태가 없는 앱을 배포할 때 사용하는 가장 기본적인 컨트롤러
- 레플리카세트를 관리하면서 앱 배포를 더 세밀하게 관리한다
- 앱을 배포할 때 롤링 업데이트를 하거나, 앱 배포 도중 잠시 뭐췄다 다시 배포할 수 있다. 또한 앱 배포 후 이전 버전으로 롤백할 수도 있다.

### 6.3.1 디플로이먼트 사용하기

```bash
# deployment/deployment-nginx.yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx-deployment
        image: nginx
        ports:
        - containerPort: 80
```

```bash
$ kubectl apply -f deployment/deployment-nginx.yaml
deployment.apps/nginx-deployment created
```

```bash
$ kubectl get deploy,rs,rc,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           3m8s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-6ffb7f48cb   3         3         3       3m8s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-6ffb7f48cb-j2xmz   1/1     Running   0          3m8s
pod/nginx-deployment-6ffb7f48cb-n9cnf   1/1     Running   0          3m8s
pod/nginx-deployment-6ffb7f48cb-tgfwv   1/1     Running   0          3m8s
```

- nginx-deployment의 컨테이너 이미지 설정 정보를 업데이트 하는 방법
    - kubectl set으로 직접 컨테이너 이미지 지정
    - kubectl edit 으로 현재 파드의 설정 정보를 연 다음 컨테이너 이미지 정보를 수정
    - 처음 적용했던 템플릿의 컨테이너 이미지 정보 수정 후 kubectl apply

```bash
$ kubectl set image deployment/nginx-deployment nginx-deployment=nginx:1.9.1
deployment.apps/nginx-deployment image updated
$ kubectl get deploy,rc,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     2            3           7h11m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-68ff66cbdf   2         2         1       16s
replicaset.apps/nginx-deployment-6ffb7f48cb   2         2         2       7h11m

NAME                                    READY   STATUS              RESTARTS   AGE
pod/nginx-deployment-68ff66cbdf-n9dpb   1/1     Running             0          16s
pod/nginx-deployment-68ff66cbdf-vzwl5   0/1     ContainerCreating   0          7s
pod/nginx-deployment-6ffb7f48cb-j2xmz   1/1     Running             0          7h11m
pod/nginx-deployment-6ffb7f48cb-tgfwv   1/1     Running             0          7h11m
```

```bash
$ kubectl edit deploy nginx-deployment 
deployment.apps/nginx-deployment edited

$ vi ./deployment/deployment-nginx.yaml 
$ kubectl get deploy,rc,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           7h15m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-68ff66cbdf   0         0         0       3m33s
replicaset.apps/nginx-deployment-6ffb7f48cb   0         0         0       7h15m
replicaset.apps/nginx-deployment-7d7598b977   3         3         3       72s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-7d7598b977-5zxj9   1/1     Running   0          64s
pod/nginx-deployment-7d7598b977-bvbwg   1/1     Running   0          56s
pod/nginx-deployment-7d7598b977-lvnx5   1/1     Running   0          72s
```

```bash
$ kubectl rollout history deploy nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

```bash
$ kubectl rollout history deploy nginx-deployment --revision=3
deployment.apps/nginx-deployment with revision #3
Pod Template:
  Labels:       app=nginx-deployment
        pod-template-hash=7d7598b977
  Containers:
   nginx-deployment:
    Image:      nginx:1.10.1
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

```bash
$ kubectl rollout undo deploy nginx-deployment # REVISION 전단계인 리비전2가 리비전 4로 변경됨
```

```bash
$ kubectl rollout undo deploy nginx-deployment --to-revision=3
deployment.apps/nginx-deployment rolled back
$ kubectl rollout history deploy nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
1         <none>
4         <none>
5         <none>
```

### 6.3.3 파드 개수 조정하기

```bash
$ kubectl scale deploy nginx-deployment --replicas=5
deployment.apps/nginx-deployment scaled
$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7d7598b977-6lncn   1/1     Running   0          4m23s
nginx-deployment-7d7598b977-bhmgw   1/1     Running   0          7s
nginx-deployment-7d7598b977-cznfw   1/1     Running   0          4m20s
nginx-deployment-7d7598b977-gkd9t   1/1     Running   0          7s
nginx-deployment-7d7598b977-xzdjw   1/1     Running   0          4m25s
```

### 6.3.4 디플로이먼트 배포 정지, 배포 재개, 재시작
- kubectl rollout 을 이용해 진행중인 배포를 잠시 멈췄다 다시 시작할 수 있다.

```bash
$ kubectl rollout pause deploy/nginx-deployment # 배포 정지

$ kubectl rollout resume deploy/nginx-deployment # 배포 재시작
```

- 배포중 작업을 하는 동안 상태가 진행 `Progressing` 으로 표시된다
- 배포가 이상없이 끝나면 배포 상태는 완료 `complete` 가 된다.
- 배포중 이상이 있으면 실패 `fail` 이 된다. (보통 다음 이유 때문에 실패함)
    - 쿼터 부족
    - readinessProbe 진단 실패
    - 컨테이너 이미지 가져오기 에러
    - 권한 부족
    - 제한 범위 초과
    - 앱 실행 조건 잘못 지정

## 6.4 데몬세트

- 클러스터 전체 노드에 특정 파드를 실행할 때 사용하는 컨트롤러

### 6.4.1 데몬세트 사용하기

- 책 참고

### 6.4.2 데몬세트의 파드 업데이트 방법 변경하기

## 6.5 스테이트풀 세트

- 상태가 있는 파드들을 관리하는 컨트롤러
- 볼륨을 사용해 특정 데이터를 저장한 후 파드를 재시작했을 때 해당 데이터를 유지한다.
- 여러개의 파드 사이에 순서를 지정해 실행되도록 할 수 있다.

### 6.5.1 스테이트풀세트 사용하기

- 기존과 다르게 파드 이름+uuid 가 아니라 `web-0`, `web-1` 과 같이 이름 뒤에 숫자가 순서대로 붙는다.
- 순서대로 실행되어야 하므로 앞 번호가 정상적으로 실행되지 않았다면 다음 번호는 실행되지 않는다.
- 삭제될 때는 반대로 큰 숫자부터 반대로 삭제된다.

### 6.5.2 파드를 순서 없이 실행하거나 종료하기

- 기본 동작은 순서대로 파드를 관리하는 것이지만 .spec.podManagementPolicy 필드를 이용해 순서를 없앨 수도 있다.

### 6.5.3 스테이트풀세트로 파드 업데이트하기

- 템플릿을 변경했을 때 자동으로 예전 파드를 삭제하고 새로운 파드 실행

## 6.6 잡

- 실행된 후 종료해야 하는 성격의 작업을 실행시킬 때 사용하는 컨트롤러
- 특정 개수만큼 파드를 정상적으로 실행 종료함을 보장해야 한다.

### 6.6.1 잡 사용하기

- 책 참고

### 6.6.2 잡 병렬성 관리

- 잡 하나가 몇 개의 파드를 동시에 실행할지를 `잡 병렬성` 이라고 한다.
- `.spec.parallelism` 필드에 설정 한다.
- 옵션을 설정할지라도 다음 이유로 지정된 값보다 잡이 파드를 적게 실행시킬 수 있다.
    - 정상 완료되는 잡의 개수를 고정하려면 병렬로 실행되는 실제 파드의 개수가 정상완료를 기다리며 남아있는 잡의 개수를 넘지 않아야 한다.
    - 워크 큐용 잡에서는 파드 하나가 정상적으로 완료되었을 때 새로운 파드가 실행되지 않는다.
    - 잡 컨트롤러가 반응하지 못할 때도 있다.
    - 잡 컨트롤러가 자원 부족이나 권한 부족 같은 이유로 파드를 실행하지 못할 때도 있다.
    - 잡이 실행시킨 파드들이 너무 많이 실패했으면 잡 컨트롤러가 새로운 파드 생성을 제한 할 수 있다.
    - 파드가 그레이스풀하게 종료되었을 때도 있다.

### 6.6.3 잡의 종류

- 단일 잡
    - 파드 하나만 실행
    - 파드가 정상적으로 실행 종료 되면 잡 실행을 완료
    - .spec.completions(정상적으로 실행 종료되어야 하는 파드 갯수), .spec.parallelism(병렬성을 지정하는 필드, 동시에 실행되어야 하는 파드 갯수) 필드 설정 x(default 1)
- 완료된 잡 갯수가 있는 병렬 잡
    - .spec.completions 양수 설정
    - .spec.parallelism 설정하지 않거나 기본값인 1로 설정
- 워크 큐가 있는 병렬 잡
    - .spec.completions 설정하지 않거나 기본값인 1로 설정
    - .spec.parallelism 양수로 설정
    - 파드 각각은 정상적으로 실행 종료되었는지 독립적으로 결정
    - 파드 하나라도 정상적으로 실행 종료되면 새로운 파드 실행되지 않는다
    - 최소한 파드 1개가 정상적으로 실행 종료되면 다른 파드는 더이상 동작하지 않거나 어떤 작업 처리 결과를 내지 않는다.

### 6.6.4 비정상적으로 실행 종료된 파드 관리

- 파드 안에 비정상적으로 실행 종료된 컨테이너가 있을 때를 대비해 컨테이너 재시작 정책을 설정하는 .spec.template.spec.restartPolicy 필드를 지정할 수 있다.
    - OnFailure : 파드가 원래 실행 중이던 노드에서 컨테이너를 재시작
    - Never : 재시작을 막는다. (ex. 장애, 업그레이등의 이유로 정지될 때)

### 6.6.5 잡 종료와 정리

- 잡이 정상적으로 실행 종료되면 파드가 새로 생성되지도 삭제되지도 않는다.
- 특정 시간을 지정해 잡 실행을 종료하려면 .spec.activeDeadlineSeconds 필드에 시간을 설정
- 잡 삭제는 kubectl delete job [jobName]
- 잡을 삭제하면 관련 파드도 같이 삭제된다.

### 6.6.6 잡 패턴

- 잡에서 파드를 병렬로 실행했을 때 파드 각각이 서로 통신하면서 동작하지 않는다.
- 잡의 일반적인 사용패턴
    - 작업마다 잡을 하나씩 생성해 사용하는 것보다는 모든 작업을 관리하는 잡 하나를 사용하는 것이 좋다. 잡 생성은 오버헤드가 크기 때문
    - 작업 개수만큼 파드를 생성하는 것보다 파드 하나가 여러 개의 작업을 처리하는 것이 좋다. 파드 생성은 오버헤드가 크므로
    - 워크 큐를 사용한다면 카프카 or RabbitMQ 같은 큐 서비스로 워크 큐를 구현하도록 기존 프로그램이나 컨테이너를 수정해야 한다.

## 6.7 크론잡

- 크론잡은 잡을 시간 기준으로 관리하도록 생성한다.
- 리눅스나 유닉스의 cron 명령어 형식 그대로 사용한다.

### 6.7.1 크론잡 사용

- 두가지 방법으로 사용 가능

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate: 
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date;echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```bash
$ kubectl run hello --schedule="*/1 * * * *" --restart=OnFailure --image=busybox -- bin/sh -c "date; echo Hello from the Kubernetes cluster"
cronjob "hello" created
```

### 6.7.2 크론잡 설정

- .spec.startingDeadlineSeconds 필드와 .spec.concurrencyPolicy가 존재한다.
- .spec.startingDeadlineSeconds
    - 지정된 시간에 크론잡이 실행되지 못했을 때 필드 값으로 설정한 시간까지 지나면 크론잡이 실행되지 않는다.
    - 없으면 시간이 지나더라도 제약 없이 잡이 실행됨
- .spec.concurrencyPolicy
    - 크론잡이 실행하는 잡의 동시성 관리 (Default ALLOW) 크론잡이 여러 개의 잡을 동시에 실행할 수 있도록 한다.
    - Forbid로 설정하면 잡을 동시에 실행하지 못하도록 한다.
    - Replace로 설정하면 이전에 실행했던 잡이 실행 중인 상태에서 새로운 잡을 실행할 시간일 때 이전에 실행 중이던 잡을 새로운 잡으로 대체한다.

- .spec.successfulJobHistoryLimit
    - Default 3, 0으로 설정시 잡이 종료된 다음에 내역을 저장하지 않습니다.
- .spec.failedJobHistoryLimit
    - Default 1, 0으로 설정시 잡이 종료된 다음에 내역을 저장하지 않습니다.
