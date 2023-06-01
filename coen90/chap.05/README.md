# 5장 파드

## 5.1 파드 개념

- k8s는 실제로 파드라는 단위로 컨테이너를 묶어서 관리하므로 보통 컨테이너 하나가 아닌 여러개의 컨테이너로 구성된다.
- 파드에 단일 컨테이너만 있을 때는 일반적인 방식으로 컨테이너를 관리해도 된다.
- 파드로 컨테이너 여러 개를 한번에 관리할 때는 컨테이너마다 역할을 부여할 수 있다.
- 파드 하나에 속한 컨테이너들은 모두 노드 하나 안에서 실행된다.
    
![스크린샷 2023-05-31 오전 7 58 25](https://github.com/Coen90/kub-introduction-study/assets/81370558/6cdda30e-97cc-4e60-bbd1-8e27b877a2be)



    
- 위 예시에서는 파드 안에 3개의 컨테이너가 있고 파드 하나 안에 있는 컨테이너들이 IP 하나를 공유한다.
- 외부에서 이 파드에 접근할 때는 그림의 192.168.10.10 이라는 IP로 접근하고, 파드 안 컨테이너와 통신할 때는 컨테이너마다 다르게 설정한 포트를 사용한다.
- 컨테이너 하나에 앞의 세가지 역할을 모두 부여할 수 있지만, 컨테이너 하나 안에 프로세스를 2개 실행하도록 설정하는것도 간단하지 않고, 관리효율도 낮다.
    
    

## 5.2 파드 사용하기

```yaml
# pod/pod-sample.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-simple-pod # 파드 이름
  labels:
    app: kubernetes-simple-pod # 오브젝트 식별 레이블 (앱 컨테이너 kubernetes-simple-pod 라는 뜻)
spec:
  containers:
  - name: kubernetes-simple-pod # 컨테이너 이름
    image: arisu1000/simple-container-app:latest # 컨테이너에서 사용할 이미지
    ports:
    - containerPort: 8080 # 해당 컨테이너에 접속할 포트 번호
```

```bash
$ kubectl apply -f ./pod/pod-sample.yaml
pod/kubernetes-simple-pod created

$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
kubernetes-simple-pod   1/1     Running   0          5s
```

## 5.3 파드 생명주기

- pod는 생성부터 삭제까지의 과정에 생명주기가 있습니다.
    - Pending: k8s시스템에 파드 생성중. 컨테이너 이미지를 다운로드한 후 전체 컨테이너 실행하는 도중이므로 파드 안의 전체 컨테이너가 실행될 때까지 시간이 걸린다.
    - Running: 파드 안 모든 컨테이너가 실행중인 상태
    - Succeeded: 파드 안 모든 컨테이너가 정상 실행 종료된 상태. 재시작되지 않는다.
    - Failed: 파드 안 모든 컨테이너 중 정상적으로 실행 종료되지 않은 컨테이너가 있는 상태. 컨테이너 종료코드가 0이 아니면 비정상 종료이거나 시스템이 직접 컨테이너를 종료한 것
    - Unknown: 파드의 상태를 확인할 수 없는 상태. 보통 파드가 있는 노드와 통신할 수 없을 때

```bash
# 현재 파드의 생명주기 -> Status 보기
$ kubectl describe pods kubernetes-simple-pod
Name:             kubernetes-simple-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             instance-5/10.182.0.6
Start Time:       Tue, 30 May 2023 23:20:41 +0000
Labels:           app=kubernetes-simple-pod
Annotations:      cni.projectcalico.org/containerID: c5d36e2bde43605fd2a9e89e6ea73de33031e54e04e32c9a3c97bf7fee7e540e
                  cni.projectcalico.org/podIP: 10.233.77.136/32
                  cni.projectcalico.org/podIPs: 10.233.77.136/32
Status:           Running # 파드의 생명주기 현재상태
IP:               10.233.77.136
IPs:
  IP:  10.233.77.136
Containers:
  kubernetes-simple-pod:
    Container ID:   containerd://b3ffd2bad8ef9e0dafd9e53d2a6fec2651e6f4dd7b010759aa2ccc1e5dbddd24
    Image:          arisu1000/simple-container-app:latest
    Image ID:       docker.io/arisu1000/simple-container-app@sha256:18f5a0fb9d1faf26862eb7a301b5c2a8debe80f60e5db03ddeb16977d3c76011
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 30 May 2023 23:20:45 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vr79c (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-vr79c:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  18m   default-scheduler  Successfully assigned default/kubernetes-simple-pod to instance-5
  Normal  Pulling    18m   kubelet            Pulling image "arisu1000/simple-container-app:latest"
  Normal  Pulled     18m   kubelet            Successfully pulled image "arisu1000/simple-container-app:latest" in 2.535347426s (2.535367594s including waiting)
  Normal  Created    18m   kubelet            Created container kubernetes-simple-pod
  Normal  Started    18m   kubelet            Started container kubernetes-simple-pod
```

- Conditions 항목은 파드의 현재 상태 정보를 나타내며 Type과 Status(true, false, unknown)으로 구분됨
    - Initialized: 모든 초기화 컨테이너가 성공적으로 시작 완료
    - Ready: 파드는 요청들을 실행할 수 있고 연결된 모든 서비스의 로드밸런싱 풀에 추가되어야 한다
    - ContainersReady: 파드 안 모든 컨테이너가 준비 상태
    - PodScheduled: 파드가 하나의 노드로 스케줄을 완료
    - Unscheduled: 스케쥴러가 자원의 부족이나 다른 제약 등으로 당장 파드를 스케쥴 할 수 없다

```bash
$ kubectl delete pod kubernetes-simple-pod 
pod "kubernetes-simple-pod" deleted
```

## 5.4 kubelet으로 컨테이너 진단하기

- 컨테이너가 실행된 후에는 kubelet가 컨테이너를 주기적으로 진단한다. 이때 필요한 프로브(Probe)에는 다음 두가지가 있다.
    - livenessProbe: 컨테이너가 실행됐는지 확인. 이 진단이 실패하면 kubelet은 컨테이너 종료. 재시작 정책에 따라 컨테이너 재시작. 컨테이너에 따로 설정하지 않으면 기본 상태 값 `Success`
    - readlinessProbe: 컨테이너가 실행된 후 실제로 서비스 요청에 응답할 수 있는지 진단. 이 진단이 실패하면 엔드포인트 컨트롤러는 해당 파드에 연결된 모든 서비스를 대상으로 엔드포인트 정보를 제거. 첫 번째 readlinessProbe 전까지의 기본 상태값은 `Failure`. readlinessProbe를 지원하지 않는 컨테이너라면 기본 상태값 `Success`
- readlinessProbe를 지원하는 컨테이너는 실행된 다음 바로 서비스에 투입되어 트래픽을 받지 않고, 실제 트래픽을 받을 준비가 되었음을 확인한 후 트래픽을 받을 수 있다. 자바와 같이 프로세스가 시작된 후 앱이 초기화 될  때 까지 시간이 걸리는 상황에 유용하다.
- 또한 앱을 실행할 때 대용량 데이터를 불러와야 하거나 컨테이너 실행은 시작 됐지만 앱의 환경설정 실수로 앱이 실행되지 않는 상황 등에 대비가 가능하다.
- 컨테이너 진단은 컨테이너가 구현한 핸들러를 kubelet이 호출해 실행하는데, 핸들러에는 세가지가 있다.
    - ExecAction: 컨테이너 안에 지정된 명령을 실행하고 종료 코드가 0일때 Success라고 진단
    - TCPSocketAction: 컨테이너 안에 지정된 IP와 포트로 TCP상태를 확인하고 포트가 열려있으면 Success라고 진단
    - HTTPGetAction: 컨테이너 안에 지정된 IP, 포트, 경로로 HTTP GET 요청을 보낸다. 응답코드가 200에서 400 사이면 Success라고 진단

## 5.5 초기화 컨테이너

초기화 컨테이너는 앱 컨테이너가 실행되기 전 파드를 초기화한다. 보안상 이유로 앱 컨테이너 이미지와 같이 두면 안되는 앱의 소스 코드를 별도로 관리할 때 유용하다.

- 초기화 컨테이너는 여러개 구성 가능. 초기화 컨테이너가 여러개 있다면 파드 템플릿에 명시한 순서대로 초기화 컨테이너가 실행된다.
- 초기화 컨테이너 실행이 실패하면 성공할 때 까지 재시작. 초기화 컨테이너의 특성을 이용하면 k8s의 선언적 이라는 특징에서 벗어날 수 있다. 필요한 명령들을 순서대로 실행하는데 사용하기 때문
- 초기화 컨테이너가 모두 실행된 후 앱 컨테이너 실행이 시작됨
- 초기화 컨테이너와 앱 컨테이너의 차이점
    - readinessProbe를 지원하지 않는다
    - 파드가 모두 준비되기 전에 실행한 후 종료되는 컨테이너이기 때문

```yaml
# ~/pod/pod-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-simple-pod
  labels:
    app: kubernetes-simple-pod
spec:
  initContainers: # 초기화 컨테이너
  - name: init-myservice
    image: arisu1000/simple-container-app:latest
    command: ['sh', '-c', 'sleep 2; echo helloworld01;'] # 이 컨테이너를 실행할 때 2초 대기 후 helloworld01 출력
  - name: init-mydb
    image: arisu1000/simple-container-app:latest
    command: ['sh', '-c', 'sleep 2; echo helloworld02;']
  containers:
  - name: kubernetes-simple-pod
    image: arisu1000/simple-container-app:latest
    command: ['sh', '-c', 'echo The app is running! && sleep 3600'] # 실행할때 The app is running! 출력하고 3600초 대기
```

## 5.6 파드 인프라 컨테이너

- k8s 모든 파드에서 항상 실행되는 pause라는 컨테이너가 있습니다. 이 pause를 `파드 인프라 컨테이너`라고 한다

![스크린샷 2023-05-31 오후 3.44.29.png](https://s3-us-we![스크린샷 2023-05-31 오후 3 44 29](https://github.com/Coen90/kub-introduction-study/assets/81370558/d1dc4150-6233-4ace-8187-db4a32933aae)


- pause는 파드 안 기본 네트워크로 실행되며 프로세스 식별자(PID)가 1로 설정되어 다른 컨테이너으 ㅣ부모 컨테이너 역할을 한다.
- 파드 안 다른 컨테이너는 pause 컨테이너가 제공하는 네트워크를 공유해서 사용
- 파드 안 다른 컨테이너가 재시작됐을 때는 파드의 IP를 유지하지만 pause 컨테이너가 재시작되면 파드 안 모든 컨테이너도 재시작한다.
- `--pod-infra-container-image` 라는 옵션이 있는데, pause가 아닌 다른 컨테이너를 파드 인프라 컨테이너로 지정할 때 사용

## 5.7 스태틱 파드

- kube-apiserver를 통하지 않고 kubelet이 직접 실행하는 파드들이 있다.
- —pod-manifest-path 라는 옵션에 지정한 디렉터리에 스태틱 파드로 실행하려는 파드들을 넣어두면 kubelet이 그걸 감지해서 파드로 실행한다
- 스태틱 파드는 kubelet이 직접 관리하면서 이상이 생기면 재실행
- kubelet이 실행중인 노드에서만 실행되고 다른 노드에서는 실행되지 않는다.
- k8s에서 파드를 실행하려면 kube-apiserver가 필요한데 kube-apiserver자체를 처음 실행하는 별도의 수단으로 스태틱 파드를 이용한다
- 스태틱 파드의 정의서는 서버 내 특정 디렉토리에 yaml 형태로 존재
- 기본 디렉토리는 /etc/kubernetes/manifest 다
- 스테틱 파드가 어느 노드에서 실행중인지 보려면 kubectl get pods --all-namespaces -o wide

## 5.8 파드에 cpu와 메모리 자원 할당

- 노드 하나에 자원 사용량이 많은 파드가 모여있다면 파드들의 성능이 나빠진다. 또한 전체 클러스터의 자원 사용 효율도 낮아진다.
- k8s에는 이러한 상황을 막는 여러 방법이 있다. 가장 기본적인 방법은 파드를 설정할 때 파드 안 각 컨테이너가 CPU나 메모리를 얼마나 사용할 수 있는지 조건을 지정하는 것
    - .spec.containers[].resources.limits.cpu
    - .spec.containers[].resources.limits.memory
    - .spec.containers[].resources.requests.cpu
    - .spec.containers[].resources.requests.memory

```yaml
# pod/pod-resource.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-simple-pod # 파드 이름
  labels:
    app: kubernetes-simple-pod # 오브젝트 식별 레이블 (앱 컨테이너 kubernetes-simple-pod 라는 뜻)
spec:
  containers:
  - name: kubernetes-simple-pod # 컨테이너 이름
    image: arisu1000/simple-container-app:latest # 컨테이너에서 사용할 이미지
    resources:
			requests: # 최소 자원 요구사항
				cpu: 0.1 # 10% 할당
				memory: 200M
			limits: # 최대 사용 제한
				cpu: 0.5 # 50% 할당
				memory: 1G
		ports:
		- containerPort: 8080
```

## 5.9 파드에 환경 변수 설정하기

- 컨테이너의 장점중 하나는 개발 환경에서 만든 컨테이너의 환경 변수만 변경해 실제 환경에서 실행하더라도 그대로 동작

```yaml
# pod/pod-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-simple-pod # 파드 이름
  labels:
    app: kubernetes-simple-pod # 오브젝트 식별 레이블 (앱 컨테이너 kubernetes-simple-pod 라는 뜻)
spec:
  containers:
  - name: kubernetes-simple-pod # 컨테이너 이름
    image: arisu1000/simple-container-app:latest # 컨테이너에서 사용할 이미지
		ports:
		- containerPort: 8080
		env:
		- name: TESTENV01 # 환경 변수 이름
			value: "testvalue01" # 값
		- name: HOSTNAME
			valueForm: # 값을 직접 할당하는 것이 아니라 어딘가 다른 곳에서 참조하는 값을 설정
				fieldRef: # 파드의 현재 설정 내용을 값으로 설정한다는 선언
					fieldPath: spec.nodeName # Ref 어디서
		- name: POD_NAME
			valueForm:
				fieldRef:
					fieldPath: metadata.name
		- name: POD_IP
			valueForm:
				fieldRef:
					fieldPath: status.podIp
		- name: POD_IP
			valueForm:
				resourceFieldRef: # 컨테이너에 CPU, 메모리 사용량을 얼마나 할당했는지 정보를 가져온다.
					containerName: kubernetes-simple-pod # 환경 변수 설정을 가져올 컨테이너 이름
					resource: requests.cpu # 어떤 자원?
		- name: CPU_LIMIT
			valueForm:
				resourceFieldRef:
					containerName: kubernetes-simple-pod
					resource: limits.cpu
```

## 5.10 파드 환경 설정 내용 적용하기

```bash
$ kubectl apply -f pod-all.yaml
pod/kubernetes-simple-pod created
```

```bash
$ kubectl exec -it kubernetes-simple-pod sh # 컨테이너 접속
$ env
# ... 환경설정 정보들
```

환경 변수 중에서 KUBERENTES_라는 prefix가 있는 아이들은 k8s 안에서 현재 사용중인 자원 관련 정보가 있다.

## 5.11 파드 구성 패턴

파드로 여러 개의 컨테이너를 묶어 구성하고 실행할 때 몇 가지 패턴을 적용할 수 있다.

### 5.11.1 사이드카 패턴

- 원래 사용하려던 기본 컨테이너의 기능을 확장하거나 강화하는 용도의 컨테이너를 추가하는 것
- 기본 컨테이너는 원래 목적의 기능에 충실
- 나머지 공통 부가 기능들은 사이드카 컨테이너를 추가해 사용
    

    <img width="375" alt="스크린샷 2023-05-31 오후 11 38 56" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/b3834666-399d-442e-9320-a025c1ec9bb5">

- 공통 역할을 하는 컨테이너의 재사용성을 높일 수 있다.

### 5.11.2 앰버서더 패턴

- 파드 안에서 프록시 역할을 하는 컨테이너를 추가하는 패턴
- 파드 안에서 외부 서버에 접근할 때 내부 프록시에 접근하도록 설정하고 실제 외부와의 연결은 프록시에서 알아서 처리
    

  <img width="211" alt="스크린샷 2023-05-31 오후 11 40 59" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/3f6fa37c-8d29-461b-9fa9-18ecf52ee96a">

    
- 웹서버 컨테이너는 캐시에 localhost로만 접근하고 실제 외부 캐시 중 어디로 접근할지는 프록시 컨테이너가 처리하는 식

### 5.11.3 어댑터 패턴

- 파드 외부로 노출되는 정보를 표준화하는 어댑터 컨테이너를 사용한다는 뜻
- 프로메테우스에서 사용중인 패턴
    
    ![스크린샷 2023-05-31 오후 11.43.22.png](https://<img width="578" alt="스크린샷 2023-05-31 오후 11 43 22" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/c72ffb2e-53c1-467f-b42f-200e52690987">
