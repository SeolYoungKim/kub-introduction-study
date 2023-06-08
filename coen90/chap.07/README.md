# 7 서비스

## 7.1 서비스의 개념

- 파드는 컨트롤러가 관리하므로 한 군데에 고정해서 실행되지 않고, 클러스터 안을 옮겨 다닌다. 이 과정에서 노드를 옮겨다니며 실행되기도 하고 클러스터 안 파드의 IP가 변경되기도 한다. 이렇게 동적으로 변하는 파드들에 고정적으로 접근할 때 사용하는 방법이 `서비스`이다.
- 서비스를 사용하면 고정주소를 이용해 파드에 접근할 수 있다.(L4 영역)
    <img width="593" alt="스크린샷 2023-06-05 오후 6 29 30" src="https://github.com/Coen90/kub-introduction-study/assets/81370558/16fa8302-588a-4d3d-9adb-40479e2c35b1">


## 7.2 서비스 타입

서비스 타입은 크게 네가지가 있다.

- ClusterIP
    - 기본 서비스 타입, k8s 클러스터 안에서만 사용할 수 있다.
    - 클러스터 IP를 이용해 서비스에 연결된 파드에 접근
    - 클러스터 외부에서 이용 불가
- NodePort
    - 서비스 하나에 모든 노드의 지정된 포트를 할당
    - node1:8080, node2:8080 처럼 노드 상관 없이 서비스에 지정된 포트 번호만 사용하면 파드에 접근할 수 있다.
    - 노드의 포트를 사용하므로 클러스터 외부에서 접근 가능
    - node2:8080이 존재하지 않으면 node1:8080으로 연결함(왜?)
- LoadBalancer
    - 퍼블릭, 프라이빗 클라우드, 쿠버네티스를 지원하는 로드밸런서 장비에서 사용
    - 클라우드 로드밸런서와 파드 연결 후 lb의 ip를 이용해 외부에서 파드에 접근
    - kubectl get service 로 확인하면 EXTERNAL-IP 항목에 lb 의 Ip가 표시됨
- ExternalName
    - 서비스를 .spec.externalName 필드에 설정한 값과 연결
    - 클러스터 안에서 외부에 접근할 때 주로 사용

## 7.3 서비스 사용하기

```yaml
# 서비스 템플릿 기본 구조
apiVersion: v1
kind: Service
metadata: 
  name: my-service
spec:
  type: ClusterIP # Default => ClusterIP
  clusterIP: 10.0.10.10 # 클러스터 IP 직접 설정 (안하면 임의값)
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

- 디플로이먼트로 생성하는 서비스에 연결할 파드 실행

```bash
$ kubectl run nginx-for-service --image=nginx --port=80 --labels="app=nginx-for-svc"
pod/nginx-for-service created
```

### 7.3.1 ClusterIP 타입 서비스 사용하기
![스크린샷 2023-06-06 오전 7 24 25](https://github.com/Coen90/kub-introduction-study/assets/81370558/995ae621-e749-4f5b-a808-dbf15d2f6ae1)

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: ckulusterip-service
spec:
  type: ClusterIP
  selector:
    app: nginx-for-svc # nginx-for-svc 파드 선택
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
$ kubectl apply -f service/clusterip.yaml 
service/clusterip-service created
$ kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
clusterip-service   ClusterIP   10.233.60.160   <none>        80/TCP    5m58s
kubernetes          ClusterIP   10.233.0.1      <none>        443/TCP   10d
```

- 더 자세한 내용을 보려면 kubectl describe service [service name]

```bash
$ kubectl get pods -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
nginx-for-service   1/1     Running   0          9h    10.233.113.147   instance-4   <none>           <none>
```

cluster IP는 k8s 클러스터 안에서만 사용할 수 있는 IP이다. 내부에서 접속 테스트를 해보자

```bash
$ kubectl run -it --image nicolaka/netshoot testnet bash # 컨테이너 bash에 접속
```

```bash
testnet:~# curl 10.233.113.147
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

### 7.3.2 NodePort 타입 서비스 사용하기

```yaml
# service/nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: nginx-for-svc
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```
![스크린샷 2023-06-06 오후 5 35 00](https://github.com/Coen90/kub-introduction-study/assets/81370558/59f88319-7dc2-432e-a545-b9aa68a4480e)

### 7.3.3 LoadBalancer 타입 서비스 사용하기

```yaml
# service/loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: nginx-for-svc
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

- LoadBalancer 타입 서비스 구조는 k8s클러스터를 외부 로드밸런서와 연계하여 설치했을 때 사용한다.
- 
![스크린샷 2023-06-06 오후 11 55 52 (1)](https://github.com/Coen90/kub-introduction-study/assets/81370558/d9ad3cfc-bb41-48fe-adf7-f6b9d2c9b8db)


```bash
$ kubectl get service
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
clusterip-service      ClusterIP      10.233.30.127   <none>        80/TCP         5m51s
kubernetes             ClusterIP      10.233.0.1      <none>        443/TCP        10d
loadbalancer-service   LoadBalancer   10.233.44.145   **<pending>**     80:31549/TCP   100s
nodeport-service       NodePort       10.233.52.37    <none>        80:30080/TCP   6h15m
```

- 지금은 외부 로드밸런서와 연결되어있지 않아 EXTERNAL-IP가 나오지 않지만 연결되어있다면 실제 외부에서 접근 가능한 IP가 나타난다.

### 7.3.4 ExternalName 타입 서비스 사용하기

```yaml
# service/externalname.yaml
apiVersion: v1
kind: Service
metadata:
  name: externalname-service
spec:
  type: ExternalName
  externalName: google.com # 연결하려는 외부 도메인 값
```

```bash
# testnet  kubectl run -it --image nicolaka/netshoot testnet bash
$ curl externalname-service.default.svc.cluster.local
<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <meta name=viewport content="initial-scale=1, minimum-scale=1, width=device-width">
  <title>Error 404 (Not Found)!!1</title>
  <style>
...
```

![스크린샷 2023-06-07 오후 5 22 36](https://github.com/Coen90/kub-introduction-study/assets/81370558/e2a43682-60c7-4b33-8e50-1dde7d55df50)

## 7.4 헤드리스 서비스

- .spec.clusterIP 필드 값을 None으로 설정하면 클러스터 IP가 없는 서비스를 만들 수 있다.
- LB가 필요없거나 단일 서비스 IP가 필요 없을때 사용
- .spec.selector 를 설정하면 k8s API로 확인할 수 있는 엔드포인트가 만들어진다.

```bash
# service/headeless.yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: nginx-for-svc
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

## 7.5 kube-proxy

- kube-proxy는 k8s에서 서비스를 만들었을 때 클러스터 IP나 노드 포트로 접근할 수 있게 만들어 실제 조작을 하는 컴포넌트
- k8s 클러스터의 노드마다 실행되면서 클러스터 내부 IP로 연결하려는 요청을 적절한 파드로 전달한다
- kube-proxy 가 네트워크를 관리하는 방법은 userspace, iptables, IPVS 가 있다.
    - userspace

        ![스크린샷 2023-06-07 오후 5 35 28](https://github.com/Coen90/kub-introduction-study/assets/81370558/ce9924cb-3bb9-4f96-a0bd-a44566b36ab3)

       
        - 클라이언트에서 서비스의 클러스터 IP를 통해 어떤 요청을 하면 iptables를 거쳐 kube-proxy가 요청을 받는다.
        - 라운드 로빈 방식 사용
    - iptables
        ![스크린샷 2023-06-07 오후 5 37 26](https://github.com/Coen90/kub-introduction-study/assets/81370558/fc8067d5-5774-4cc0-8630-4d9502118415)

        
        - kube-proxy는 iptables를 관리하는 역할만 하고 직접 클라이언트 트래픽을 받지 않는다.
        - 클라이언트 요청은 iptables를 거쳐 파드로 직접 전달된다.
    - IPVS
        ![스크린샷 2023-06-07 오후 5 37 33](https://github.com/Coen90/kub-introduction-study/assets/81370558/b5560723-e06a-4a67-8a91-695c6891cc93)
        
        - 리눅스 커널에 있는 L4 LB기술이다.
        - 다양한 LB 알고리즘을 사용할 수 있다.
