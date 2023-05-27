# 4. 쿠버네티스 아키텍처 
## 1) 쿠버네티스 클러스터의 전체 구조
![img.png](img.png)
- 쿠버네티스 클러스터의 구성
  - 마스터 : 클러스터를 관리
  - 노드 : 실제 컨테이너를 실행 

### 마스터 
- 실행되는 컴포넌트 리스트
  - etcd
  - kube-apiserver
  - kube-scheduler
  - kube-controller-manager
  - kubelet
  - kube-proxy
  - docker
  - etc


- 고가용성
  - 서버 3대 정도를 구성해서 운영하는 것이 일반적
  - 리더 마스터 1대 + 나머지 2대는 대기 
  - 리더 마스터 장애 -> 나머지 2대 중 1대가 리더 
  - 5대로 구성하기도


### 노드 
- 실행되는 컴포넌트 리스트
  - kubelet
  - kube-proxy
  - docker
  - etc


-  