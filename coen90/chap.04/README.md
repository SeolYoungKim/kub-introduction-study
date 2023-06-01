# 4장 쿠버네티스 아키텍처

## 4.1 쿠버네티스 클러스터의 전체 구조

쿠버네티스 클러스터는 크게 두종류의 서버로 구성하는데, 클러스터를 관리하는 `마스터`와 실제 컨테이너를 실행시키는 `노드`
<img width="588" alt="스크린샷 2023-05-30 오전 10 14 23" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/ee51a641-1f61-41ea-8a41-90f322937e8b">

### 마스터

- 마스터에는 `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `kuberet`, `kube-proxy`, `docker`등의 컴포넌트가 실행됨
- 마스터는 보통 고가용성을 위해 3대정도 구성한다.(평소 리더마스터 1대 나머지 2대는 대기)

### 노드

- `kubelet`, `kube-proxy`, `docker` 등의 컴포넌트가 실행됨.
- 실제 사용하는 컨테이너 대부분이 노드에서 실행된다.
- 초기에는 Minion이라고 불렀고, 가끔 다른 문서에서 미니언이라고 언급하는 경우가 있다고 함

<img width="620" alt="스크린샷 2023-05-30 오전 10 16 19" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/8dc2bbb8-1268-48e6-b321-957f189c141f">

## 4.2 쿠버네티스의 주요 컴포넌트

### 4.2.1 마스터용 컴포넌트

`etcd`

- 코어os에서 개발한 고가용성을 제공하는 key-value 저장소.
- 쿠버네티스에서 필요한 모든 데이터를 저장하는 DB역할을 한다.
- etcd는 서버 하나당 프로세스 1개만 사용할 수 있다.
- 보통 etc 자체를 클러스터링 한 후 여러개의 마스터 서버에 분산해서 실행해 데이터의 안정성을 보장하도록 구성
- 더 안정적으로 운영하려면 주기적으로 etcd에 있는 데이터 백업 권고

`kube-apiserver`

- 쿠버네티스 클러스터의 API를 사용할 수 있도록 하는 컴포넌트
- 클러스터로 온 요청이 유효한지 검증
    - ex) k8s API 스펙이 맞춰 조회 요청을 받으면, 요청에 사용된 토큰이 요청을 실행할 권한이 있는지 검사하고 권한이 있다면 요청을 조회하여 돌려줍니다.
- k8s는 MSA 이므로 다수의 서로 분리된 컴포넌트로 구성되어 있음. k8s에 보내는 모든 요청은 `kube-apiserver`를 이용해 다른 컴포넌트로 전달한다.

`kube-scheduler`

- 현재 클러스터 안에서 자원 할당이 가능한 노드 중 알맞은 노드를 선택해 새롭게 만든 파드를 실행한다
- 파드는 처음 실행할 때 여러 조건을 설정하며, `kube-scheduler`가 조건에 맞는 노드를 찾는다.
- 여러 조건
    - 하드웨어 요구사항
    - 함께 있어야 하는 파드들을 같은 노드에 실행하는 어피니티(affinity) 만족 여부
    - 파드를 다양한 노드로 분산해 실행하는 안티 어피니티(anti-affinity) 만족 여부
    - 특정 데이터가 있는 노드에 할당 등

`kube-controller-manager`

- k8s는 파드들을 관리하는 컨트롤러를 가지고 있음
- 컨트롤러 각각은 논리적으로 개별 프로세스이지만 복잡도를 줄이기 위해 모든 컨트롤러를 바이너리 파일 하나로 컴파일해 단일 프로세스로 실행
- k8s는 Golang으로 개발되었고, 클러스터 안에서 새로운 컨트롤러를 사용할 때는 컨트롤러에 해당하는 구조체를 만든다.
- 이 구조체를 `kube-controller-manager`가 관리하는 큐에 넣어 실행하는 방식으로 동작한다.

`cloud-controller-manager`

- k8s의 컨트롤러들을 클라우드 서비스와 연결해 관리하는 컴포넌트
- 각 클라우드 서비스에서 직접 관리하는데, 보통 네가지 컴포넌트를 관리한다.
    - 노드 컨트롤러: 클라우드 서비스 안에서 노드를 관리하는데 사용
    - 라우트 컨트롤러: 각 클라우드 서비스 안의 네트워크 라우팅을 관리하는데 사용
    - 서비스 컨트롤러: 각 클라우드 서비스에서 제공하는 로드밸런서를 생성, 갱신, 삭제하는데 사용
    - 볼륨 컨트롤러: 클라우드 서비스에서 생성한 볼륨을 노드에 연결하거나 마운트하는 등에 사용

## 4.2.2 노드용 컴포넌트

- k8s 실행환경을 관리. 대표적으로 각 노드의 파드 실행을 관리

`kubelet`

- 클러스터 안 모든 노드에서 실행되는 에이전트이며 파드 컨테이너들의 실행을 직접 관리한다.
- 파드스펙 이라는 조건이 담긴 설정을 전달받아 컨테이너를 실행하고, 컨테이너가 정상적으로 실행되는지 헬스체크를 진행
- 노드안에 있는 컨테이너라도 k8s가 만들지 않은 컨테이너는 관리하지 않음

`kube-proxy`

- k8s는 클러스터 안에 별도의 가상 네트워크를 설정하고 관리하는데, 이러한 가상 네트워크의 동작을 관리하는 컴포넌트
- 호스트의 네트워크 규칙을 관리하거나 연결을 전달

`컨테이너 런타임`

- 실제로 컨테이너를 실행시킨다.
- ex) Docker, containerd, runc
- 컨테이너 표준을 정하는 OCI 의 런타임 규격을 구현한 컨테이너 런타임이라면 k8s에서 사용 가능.

## 4.2.3 애드온

- 클러스터 안에서 필요한 기능을 실행하는 pod
- 네임스페이스는 kube-system 이며 애드온으로 사용하는 파드들은 deployment, replication controller등으로 관리

`네트워킹 애드온`

- 클러스터 안에 가상 네트워크를 구성해 사용할 때 사용
- 퍼블릭 클라우드에서 제공하는 k8s는 별도의 네트워킹 애드온을 제공한다
- 직접 서버를 구성한다면 네트워킹 관련 애드온을 설치해서 사용해야한다.
- k8s를 직접 서버에 구성할 때 가장 까다로운 부분이다.

`DNS 애드온`

- 클러스터 안에서 동작하는 DNS 서버
- k8s 안에 실행된 컨테이너들은 자동으로 DNS 서버에 등록된다
- kube-dns 와 coreDNS 가 있는데, 1.13 부터는 CoreDNS가 기본 DNS 애드온이다.

`대시보드 애드온`

- 웹UI이다

`컨테이너 자원 모니터링`

- 클러스터 안에서 실행중인 컨테이너의 상태를 모니터링하는 애드온
- CPU, 메모리 사용량 같은 데이터들을 시계열 형식으로 저장해 볼 수 있다.
- kubelet에 포함된 cAdvisor라는 컨테이너 모니터링 도구를 사용

(https://s3-us-west-2.amazonaws.com/secure.notion-static.com/640d635d-cded-4f82-a32c-089690a7f589/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7_2023-05-30_%EC%98%A4%ED%9B%84_3.21.50.png)
    

`클러스터 로깅`

- 클러스터 안 개별 컨테이너의 로그와 k8s 구성 요소의 로그들을 중앙화한 로그 수집 시스템에 모아서 보는 애드온
- 클러스터 안 각 노드에 로그를 수집하는 파드를 실행해서 중앙 저장 파드로 로그를 수집
- 로그를 수집해서 보여줄 때는 ELK나 EFK를 많이 사용한다.

![스크린샷 2023-05-30 오후 3.21.57.png](https://s3-us-west<img width="609" alt="스크린샷 2023-05-30 오후 3 21 57" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/4d6d6667-5972-41ab-ad29-4f018b17c4c0">


## 4.3 오브젝트와 컨트롤러

- k8s는 크게 오브젝트와 오브젝트를 관리하는 컨트롤러로 나눈다.
- 오브젝트
    - pod
    - service
    - volume
    - namespace 등
- 컨트롤러
    - ReplicaSet
    - Deployment
    - StatefulSet
    - DaemonSet
    - Job 등

### 4.3.1 네임스페이스

- k8s 클러스터 하나를 여러개의 논리적 단위로 나누어 사용하는 것
- 네임스페이스 덕분에 k8s 클러스터 하나를 여러개 팀이나 사용자가 함께 공유할 수 있다
- 네임스페이스별로 별도의 쿼터를 설정해서 특정 네임스페이스의 사용량을 제한 가능

```bash
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   3d12h
kube-node-lease   Active   3d12h
kube-public       Active   3d12h
kube-system       Active   3d12h
```

- NAME
    - default: 기본 네임스페이스. 명령을 실행할 때 별도의 네임스페이스를 지정하지 않는다면 default 네임스페이스에 명령 적용
    - kube-system: k8s 시스템에서 관리하는 네임스페이스. 쿠버네티스 관리용 파드나 설정이 있다
    - kube-public: 클러스터 안 모든 사용자가 읽을 수 있는 네임스페이스. 클러스터 사용량 같은 정보를 관리
    - kube-node-lease: 각 노드의 임대 오브젝트들을 관리하는 네임스페이스
- kubectl로 네임스페이스를 지정해서 사용할 때는 `--namespace=kube-system`과 같이 네임스페이스를 명시해야 한다.
- 기본 네임스페이스를 변경하려면 context 정보를 확인해야 한다.

```bash
$ kubectl config current-context 
kubernetes-admin@cluster.local

$ kubectl config get-contexts kubernetes-admin@cluster.local # context 정보 확인 | 네임스페이스가 비어있다.
CURRENT   NAME                             CLUSTER         AUTHINFO           NAMESPACE
*         kubernetes-admin@cluster.local   cluster.local   kubernetes-admin

# kubernetes-admin@cluster.local 네임스페이스 변경
$ kubectl config set-context kubernetes-admin@cluster.local --namespace=kube-system
Context "kubernetes-admin@cluster.local" modified.

$ kubectl config get-contexts kubernetes-admin@cluster.local 
CURRENT   NAME                             CLUSTER         AUTHINFO           NAMESPACE
*         kubernetes-admin@cluster.local   cluster.local   kubernetes-admin   kube-system

# 기본 네임스페이스를 다시 default로 변경
$ kubectl config set-context kubernetes-admin@cluster.local --namespace=default # 혹은
$ kubectl config set-context kubernetes-admin@cluster.local --namespace=
Context "kubernetes-admin@cluster.local" modified.
```

- 간단하게 네임스페이스 변경을 할 수 있도록 도와주는 kubens 라는 도구가 있다.
- wget https://raw.githubusercontent.com/ahmetb/kubectx/master/kubens

## 4.3.2 템플릿

- k8s 클러스터의 오브젝트나 컨트롤러가 어떤 상태여야 하는지를 적용할 때는 YAML 형식의 템플릿을 사용
- 템플릿의 기본 형식은 다음과 같다
    
    ```yaml
    ---
    apiVersion: v1 # 사용하려는 k8s API 버전. kubectl api-versions 로 현재 클러스터에서 사용 가능한 API 버전 확인 가능
    kind: Pod # 어떤 종류의 오브젝트 or 컨트롤러의 작업인지 명시. Pod Deployment, Ingress 등 다양한 설정
    metadata: # 메타데이터 설정. 해당 오브젝트 이름이나 레이블 등 설정
    spec: # 파드가 어떤 컨테이너를 갖고 실행하며 실행할 때 어떻게 동작해야 하는지 명시
    ```
    
- 어떤 필드가 있고 어떤 역할을 하는지는 kubectl explain 명령으로 확인 가능
    
    ```bash
    $ kubectl explain pods
    KIND:     Pod
    VERSION:  v1
    
    DESCRIPTION:
         Pod is a collection of containers that can run on a host. This resource is
         created by clients and scheduled onto hosts.
    
    FIELDS:
       apiVersion   <string>
         APIVersion defines the versioned schema of this representation of an
         object. Servers should convert recognized schemas to the latest internal
         value, and may reject unrecognized values. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
    
       kind <string>
         Kind is a string value representing the REST resource this object
         represents. Servers may infer this from the endpoint the client submits
         requests to. Cannot be updated. In CamelCase. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
    
       metadata     <Object>
         Standard object's metadata. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
    
       spec <Object>
         Specification of the desired behavior of the pod. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
    
       status       <Object>
         Most recently observed status of the pod. This data may not be up to date.
         Populated by the system. Read-only. More info:
         https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
    ```
