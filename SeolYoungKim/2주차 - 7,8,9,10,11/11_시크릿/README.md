# 11. 시크릿 
> 비밀번호, OAuth 토큰, SSH 키와 같은 민감한 정보들을 저장하는 용도로 사용 
> - 컨테이너 안에 저장하지 않고, 별도로 보관했다가 실제 파드를 실행할 때 템플릿으로 컨테이너에 제공 


## 1) 시크릿
- built-in 시크릿과 사용자 정의 시크릿이 있음 
  - built-in: 쿠버네티스 클러스터 안에서 쿠버네티스 API에 접근할 때 사용 
    - 클러스터 안에서 사용하는 ServiceAccount를 생성하면 자동으로 관련 시크릿 생성 
      - ServiceAccount가 사용 권한을 갖는 API에 접근할 수 있음 
  - 사용자 정의: 사용자가 만든 시크릿 

### 시크릿의 타입 
- Opaque: 기본값. 키-값 형식으로 임의의 데이터를 설정 
- kubernetes.io/service-account-token: 쿠버네티스 인증 토큰을 저장 
- kubernetes.io/dockerconfigjson: 도커 저장소 인증 정보를 저장 
- kubernetes.io/tls: TLS 인증서 저장

---

## 2) 시크릿 사용 
### 파드의 환경 변수로 시크릿 사용 
- `.spec.template.spec.containers[].env[]` 필드에 설정 

### 볼륨 형식으로 파드에 시크릿 제공 
- `.spec.template.spec.containers[].volumeMounts[]`, `.spec.template.spec.containers[].volumes[]`필드를 설정

### 프라이빗 컨테이너 이미지를 가져올 때 시크릿 사용 
- 인증 정보를 시크릿에 설정하여 저장한 후 사용 
  - `docker-registry`

### 시크릿으로 TLS 인증서를 저장해 사용 
- HTTPS 인증서를 저장하는 용도로써 시크릿 사용 


---

## 3) 시크릿 데이터 용량 제한 
- 시크릿 데이터는 etcd에 암호화하지 않은 텍스트로 저장됨 
  - kube-apiserver나 kubelet의 메모리 용량을 많이 차지하게 됨 
- 개별 시크릿 데이터의 최대 용량은 1MB
- etcd는 접근을 제한할 것 