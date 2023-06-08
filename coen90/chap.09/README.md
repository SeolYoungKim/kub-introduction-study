# 9 레이블과 애너테이션

- Label은 key-value 이며, 클러스터 안에 오브젝트를 만들 때 메타데이터로 설정 가능하다.
- Label key는 k8s 안에서 컨트롤러들이 파드를 관리할 때 자신이 관리해야 할 파드를 구분하는 역할
- k8s는 레이블만으로 관리대상을 구분. 특정 컨트롤러가 만든 파드라도 레이블을 변경하면 인식 불가능하다.
- 이러한 특징을 사용해 이슈가 발생한 파드만 서비스 영향 없이 따로 분리해 디버깅이 가능하다.
- 레이블 네이밍 규칙
    - 63자 이하
    - 시작과 끝 [a-z0-9A-Z]이어야한다
    - 중간에는 - _ . 등이 올 수 있다
- 레이블 키 앞에 접두어 사용 가능(보통 사용자는 접두어 사용 안함)
    - k8s시스템에서 사용하는 레이블은 [kubernetes.io/](http://kubernetes.io/) 라는 접두어 사용
- 특정 Label을 선택할 때는 레이블 셀렉터를 사용한다.
    - 등호기반 셀렉터
        - =, == 같다
        - ≠ 다르다
    - 집합기반 셀렉터
        - environment in (develop, stage) | environment = develop ||  environment = stage
        - release not in (latest, canary) | release != latest && release != canary
        - gpu
        - !gpu
- 파드에서 레이블 옵션으로 검색하는 방법
    - kubectl get pods -l app=nginx
    - kubectl get pods -l environment=develop,release=stable
    - kubectl get pods -l “app=nginx, environment not in (develop)”

## 9.2 애너테이션

- 레이블처럼 key-value 구조
- 쿠버네티스 시스템이 필요한 정보들을 담았고, k8s 클라이언트나 라이브러리가 자원을 관리하는데 활용
- 그래서 애너테이션 키는 k8s 시스템이 인식할 수 없는 값을 사용

```yaml
# annotaion/annotion.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: annotaion
  labels: 
    app: nginx
  annotations:
    manager: "myadmin"
    contact: "010-0000-0000"
    release-version: "v1.0"
spec:
  replicas: 1
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
        image: nginx
        ports: 
        - containerPort: 80
```

## 9.3 레이블을 이용한 카나리 배포

- 배포 방법
    - 롤링업데이트 : 배포된 전체 파드를 한꺼번에 교체하는 것이 아닌 일정 갯수씩 교체하면서 배포 (deployment의 기본 배포 방식)
    - 블루/그린: 기존에 실행된 파드 개수와 같은 개수의 신규 파드를 모두 실행 후 health check. 그 후 트래픽을 한꺼번에 신규 파드쪽으로 옮김
    - 카나리: 기존 버전을 유지한 채, 일부 버전만 신규 파드로 교체. 버그나 이상은 없는지, 사용자 반응은 어떤지 확인할 때 유용

```yaml
# canary/deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-testapp
  labels: 
    app: myapp
    version: stable
spec:
  replicas: 2
  selector:
    matchLabels: 
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:v0.1
        ports: 
        - containerPort: 8080
```

```yaml
# canary/deployment-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-testapp-canary
  labels: 
    app: myapp
    version: canary
spec:
  replicas: 1
  selector:
    matchLabels: 
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:v0.2
        ports: 
        - containerPort: 8080
```

```yaml
# canary/myapp-svc.yaml
apiVersion: v1
kind: Service
metadata:
  labels: 
    app: myapp
  name: myapp-svc
  namespace: default
spec:
  ports:
  - nodePort: 30880
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: myapp
  type: NodePort
```

```bash
$ curl localhost:30880 # 요청을 보내면 stable과 canary가 번갈아 응답한다
```

- canary버전이 제대로 동작하지 않는다면 디플로이먼트를 삭제하거나 .spec.replicas 필드값을 0으로 설정해 서비스에서 제외한다.
- canary 버전이 정상 동작한다면 stable 버전에 있는 컨테이너 이미지를 업데이트해 전체 서비스에 적용
