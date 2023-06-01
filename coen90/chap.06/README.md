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
