# 11 시크릿

## 11.1 시크릿 만들기

내장 시크릿

- k8s 클러스터 안에서 k8s API 에 접근할 때 사용
- 클러스터 안에서 사용하는 ServiceAccount라는 계정을 생성하면 자동으로 관련 시크릿을 생성
- 해상 시크릿으로 ServiceAccount가 사용 권한을 갖는 API에 접근 가능

사용자 정의 시크릿

- 사용자가 만든 시크릿

### 11.1.1 명령으로 시크릿 만들기

kubectl create secret으로 시크릿 생성하기

```bash
$ echo -n 'username' > ./username.txt
$ echo -n 'password' > ./password.txt
$ kubectl create secret generic user-pass-secret --from-file=./username.txt --from-file=./password.txt
secret/user-pass-secret created
$ kubectl get secret user-pass-secret -o yaml
apiVersion: v1
data:
  password.txt: cGFzc3dvcmQ= # base64
  username.txt: dXNlcm5hbWU= # base64
kind: Secret
metadata:
  creationTimestamp: "2023-06-07T15:11:09Z"
  name: user-pass-secret
  namespace: default
  resourceVersion: "735598"
  uid: 6fdb4ffb-3760-494d-b621-fe6af169ca41
type: Opaque
```

### 11.1.2 템플릿으로 시크릿 만들기

```yaml
# secret/user-pass-yaml.yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-pass-yaml
type: Opaque
data:
  username: dXNlcm5hbWU=
  password: cGFzc3dvcmQ=
```

- .type 시크릿 타입
    - Opaque: Default. key-value 형식으로 임의의 데이터 설정 가능
    - kubernetes.io/service-account-token: k8s 인증 토큰 저장
    - kubernetes.io/dockerconfigjson: 도커 저장소 인증 정보를 저장함
    - kubernetes.io/tls: TLS 인증서 저장
- 주의할점은 base64 인코딩값을 설정해야 한다는 점 (echo -n "username" | base64)

## 11.2 시크릿 사용하기

### 11.2.1 파드의 환경 변수로 시크릿 사용하기

```yaml
# secret/deployment-secret01.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secretapp
  labels:
    app: secretapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretapp
  template:
    metadata:
      labels:
        app: secretapp
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: SECRET-USERNAME
          valueFrom:
            secretKeyRef:
              name: user-pass-yaml
              key: username
        - name: SECRET-PASSWORD
          valueFrom:
            secretKeyRef:
              name: user-pass-yaml
              key: password
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: secretapp
  name: secretapp-svc
  namespace: default
spec:
  ports:
  - nodePort: 30900
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: secretapp
  type: NodePort
```

- 여기서 사용하는 시크릿은 미리 만들어져 있어야 하며, 오타 등의 실수로 없는 시크릿을 참조하거나 시크릿이 생성되어있지 않으면 에러가 발생해 파드가 실행되지 못한다.
- kubelet은 계속해서 시크릿을 로딩하고 있으므로 시크릿을 만들어주면 정상적으로 실행된다.

### 12.2.2 볼륨 형식으로 파드에 시크릿 제공하기

```yaml
# secret/deployment-secret02.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secretapp
  labels:
    app: secretapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretapp
  template:
    metadata:
      labels:
        app: secretapp
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: volume-secret
          mountPath: "/etc/volume-secret"
          readOnly: true
      volumes:
      - name: volume-secret
        secret:
          secretName: user-pass-yaml
```

### 11.2.3 프라이빗 컨테이너 이미지를 가져올 때 시크릿 사용하기

- 보통 컨테이너 이미지를 pull 할 때 대부분 공개된 이미지를 사용한다.
- 프라이빗 컨테이너 이미지를 사용할 수 있는데 인증정보가 필요하다.
- 인증정보를 시크릿에 설정해 저장한 후 사용한다.
- k8s에는 kubectl create secret 의 하위 명령으로 도커 컨테이너 이미지 저장소용 시크릿을 만드는 docker-registry가 있다.

```bash
$ kubectl create secret docker-registry dockersecret --docker-username=USERNAME --docker-password=PASSWORD --docker-email=EMAIL --docker-server=https://index.docker.io/v1/
secret/dockersecret created
$ kubectl get secrets dockersecret -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOnsidXNlcm5hbWUiOiJVU0VSTkFNRSIsInBhc3N3b3JkIjoiUEFTU1dPUkQiLCJlbWFpbCI6IkVNQUlMIiwiYXV0aCI6IlZWTkZVazVCVFVVNlVFRlRVMWRQVWtRPSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: "2023-06-07T15:52:28Z"
  name: dockersecret
  namespace: default
  resourceVersion: "742095"
  uid: 77235a1e-3448-4988-8100-e3b32454fbbb
type: kubernetes.io/dockerconfigjson
```

- 상기 .dockerconfigjson..dockerconfigjson 을 decode 해보면 아래와 같다

```json
{
    "auths":{
        "https://index.docker.io/v1/":{
            "username":"USERNAME",
            "password":"PASSWORD",
            "email":"EMAIL",
            "auth":"VVNFUk5BTUU6UEFTU1dPUkQ="
        }
    }
}
```

- yaml 형식으로 dockersecret 사용

```yaml
# secret/deployment-secret03.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secretapp
  labels:
    app: secretapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretapp
  template:
    metadata:
      labels:
        app: secretapp
    spec:
      containers:
      - name: testapp
        image: arisu1000/private-test:latest
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: dockersecret
```

- 여기서 dockersecret 값이 잘못되어있어 이미지 pull이 안된다. 올바로 설정해 위 명령어를 다시 작성하면 이미지가 pull 될것이다(아마두..)

### 11.2.4 시크릿으로 TLS 인증서를 저장해 사용하기

- 인증서 키와 crt 파일을 만든다

```yaml
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=example.com"

$ kubectl create secret tls tlssecret --key tls.key --cert tls.crt

$ kubectl get secret tlssecret -o yaml
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUREVENDQWZXZ0F3SUJBZ0lVQTR1S0FUaGZpaldWMkVSR3ZOTHREZG5NVExRd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0ZqRVVNQklHQTFVRUF3d0xaWGhoYlhCc1pTNWpiMjB3SGhjTk1qTXdOakEzTVRZd05EVTNXaGNOTWpRdwpOakEyTVRZd05EVTNXakFXTVJRd0VnWURWUVFEREF0bGVHRnRjR3hsTG1OdmJUQ0NBU0l3RFFZSktvWklodmNOCkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFKOHBVMnk0MHVKMHZaN0Vmdkgvb1pKZE9nK3UyWCthd2xBdTBFYk0KQkFuL0dxeDR0ZkFMM3Ftb1BJSm84dE5GMlhINjIyL0xYc29aY3VmckZ4Yk1OU2dvN0grbWlZT2JUMkxsaStXNQoxbnpRZXFTMUlXZ3FNV0duWHRTZkF6R1hsYnVwZjFlaDRWWmNrT1N0alpOWTNPTjZXd0JOKzdKQ3lEbHRxUnhFClc0eGRaSzNIRW9jRFFUOWdleGw4a1B4elhxS09XSEFnRTY2UmlYMVJKUkt5YXV5TDUra3FOSno1eDBjMHIyVk8KdVQ1SU5XOHBHL3U0bmJlQXBsbEROcGJpM2VqZnJoWmNEdFIwczhkcHFvQ3NVY2szelNWa21SUTNVODlZc2FIYQo3Zk9xaWlDcFcxN2VwaVVEMVdHNDlZcDFDYnJ4eVFzeHJSejRhaGE3WXhBb0VVOENBd0VBQWFOVE1GRXdIUVlEClZSME9CQllFRlBhTmV2VDJOT1FOc0ExYXNkRmw2MW5SSkQ0dk1COEdBMVVkSXdRWU1CYUFGUGFOZXZUMk5PUU4Kc0ExYXNkRmw2MW5SSkQ0dk1BOEdBMVVkRXdFQi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQgpBSkx3N1F1YThIekR6dndNSTR3Z1hNSHFtd1NCMGRYcW02Q0pLVUNFam8rZXhTRVpHbDZZSUJ6cXVvaWJhZDY3CmFxRXR2NTFrSE52SVQrZE50aFlPWXpaSTA1aHRvS3U2SjNSZjRXakFZZWV0eUt5SFZaMCtKYW5IdXdjQnRiWWYKRlVzMWJCNkxybWdFRnJjWExZU3hPc1FDQ1NIRlRzaDd4NFA1YkcwL21IeEZzS3hXT2xvazhra01BaFRPUzI5aQp0Yk41akQ0QzF6ZW1vTjcxempDUFltV3RiWlNLR1JEMnl5MnNTZGgvUVNXdlpBYWF1UlBBcjI0RzhZNU96cjFjCndpV0RzcW0wWmJ1Zy9xMUlWTlJNQ29nUnNwaVQ0MDB2Q1lNRm8zaDNoWVdESlRQSkVuK0hwSWt1S21Na1Q2aXkKYXZ6WGV6TVRZTjEwWkI2Q05oYVh3SWs9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2d0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktrd2dnU2xBZ0VBQW9JQkFRQ2ZLVk5zdU5MaWRMMmUKeEg3eC82R1NYVG9QcnRsL21zSlFMdEJHekFRSi94cXNlTFh3Qzk2cHFEeUNhUExUUmRseCt0dHZ5MTdLR1hMbgo2eGNXekRVb0tPeC9wb21EbTA5aTVZdmx1ZFo4MEhxa3RTRm9LakZocDE3VW53TXhsNVc3cVg5WG9lRldYSkRrCnJZMlRXTnpqZWxzQVRmdXlRc2c1YmFrY1JGdU1YV1N0eHhLSEEwRS9ZSHNaZkpEOGMxNmlqbGh3SUJPdWtZbDkKVVNVU3NtcnNpK2ZwS2pTYytjZEhOSzlsVHJrK1NEVnZLUnY3dUoyM2dLWlpRemFXNHQzbzM2NFdYQTdVZExQSAphYXFBckZISk44MGxaSmtVTjFQUFdMR2gydTN6cW9vZ3FWdGUzcVlsQTlWaHVQV0tkUW02OGNrTE1hMGMrR29XCnUyTVFLQkZQQWdNQkFBRUNnZ0VCQUkrbE1KSFRUU0RzMDZaVEdXODNrNDhSYkxGeTBRR0ZueEhXN2txM0huNFgKS3UrMkVoNFAyR211V000cUFkNEVFSGY2TzhudDlpTFlUUWhhK0grdTFkcms5RzFRMUpOZXZJczVPTVdncjUvKwpXSElHdDV2WFdMSVY2RlJsSHZESEtuQXdUYW05aEMzNVpSdStOeVJnOHhxcTl5NlRNekp6YTFuSlN2TWtEcXZpCndacU83amo1UFE3L3Y4cWdsRWZDQzFUYW44YkZMck9IUWdZTDF0VzAwSnpWc1RyajhzUzVmNjhoc0tLaXZUU04KcS9SeEx2c2NHbEdmbU9jMEZIWmhnVXBYdXB2Z2s0QW1zN1JYRXNBK002OVBPNmFsMW9SL00vTnpoR1JRVXFpTQpmQjd3SWtpd0lJQ1kyY1lJRGZZaEh5YzJONldYMHB0c200U1ZqR1RiRnVFQ2dZRUEwejVMaXZiY3gxQ3RwN0p6CndLcFVEb2hVZWViUm5HMGZDTHlMNUtqaUluc21ZQ1BBekN0NGRXb3FIaEcveU9yUVBhRERKR2ZxMWxEMFFocE4KWEdvRWx6YkRyNEZwTG16cmFkcTVjM09nM0Y3Q01uOE02azJDV2RqUW9uMGdnK1BhNGhLYWtFTXAwcW1BWjY0UQp5UjBCYkI0OWt5U202YUdVaXpEc1FzY01wVlVDZ1lFQXdPSWtGS2tRdVJaZ0U4Y0Rhcjh5eHE0NDRUYTR3aHloCjVkbzJPcTBiMi8yRGw5Nkc3NWxrV3F3N08vQ1JjeVJ6NWJrTURHY3RHR3BtbXRIMDRscDZhMGNZcXBWUTJmN2oKMVhNL0ZPNVVDSEhGRzR6d1NaTVMxOGJTRGJDZ240VGRYWFUwWDZEdTFjeVpiQ1lEOG5rZnpOK3lpVUxXV1dCOQphVVY1THIrWW5CTUNnWUVBdVBBdzh0aHRNWkpRZGlDbGRtZW9iNUNyWkkzUHRVTlRpREtKeHdhVDg5d2RITTR3ClhJOHlScGxMaGtmRHdBTFRqU0RSdDIzREN4NlV1Y3FOTC9zaFNjR0lVSDdidHVsa3NLZnM5RWFtN2tlSGZPMysKUUtMYkhBM1ZtbXd4cTBZd3V2dk9sYjQzUDFkbU0xOFJFd0Z4M1ZZY1VsWWtTeVpMQmhFdXhzZTlLb0VDZ1lCKwozb0JEQXExWVFPcHpOOHo4a3NTd1FIcHpVSTRZUjhNSnNBMUpiUUhOSXFSQzZZQ3g2cUJDcjlUS2FVTVNqR0NiCk1xdEZJVHhkT2VkQllHYUYyR043V3FsVDBxRDZzcGhqbHNsZ1dCNzM2dlZ1V0xiWWZoKy94Q3Y0Q3p5cmtEWVcKdWZmNENwL3VDd1REU1FJQnBFQVJmdll0S01SYXg0ZldEWGRYRTNrcTl3S0JnUURDeDc5bUhEQ3ZGeFM0N0NNYwpXRXBMc1RsalJpOEZZR01pVkc1WjRzL1prNFQrZGdDdHBTZGJ4cmFZZ3ppQ1F2c0VkWkZ0Rkd4RDQweVdKUXNLCndWSDJlcW5Yc2VENmZQWll0YmdSNDUzRmVKVy9YdktTNWQrV0lzb3JmYzdtRGh0MXJ5N29CWVplWnZsUVQ5clUKeWMrdDJVd3VNV1hnTUtsRVh5VU9jOFpQTnc9PQotLS0tLUVORCBQUklWQVRFIEtFWS0tLS0tCg==
kind: Secret
metadata:
  creationTimestamp: "2023-06-07T16:05:05Z"
  name: tlssecret
  namespace: default
  resourceVersion: "744106"
  uid: e898f2ed-c1d4-446b-86fc-02f754c3421f
type: kubernetes.io/tls
```

- .data.tls.crt와 data.tls.key 필드값이 생성되었고, secret type 은 [kubernetes.io/tls](http://kubernetes.io/tls) 로 설정되어있다.
- 이 시크릿을 인그레스와 연결해서 사용할 수 있당

## 11.3 시크릿 데이터 용량 제한

- etcd에 암호화하지 않은 텍스트로 저장된다.
- 시크릿 용량이 너무 크면 k8s kube-apiserver나 Kubelet의 메모리를 많이 차지한다
- 개별 시크릿 데이터의 최대 용량은 1MB
- 전체 시크릿 데이터 용량에 제한을 두는 기능도 도입 예정
- 중요한 서비스에 쿠버네티스를 사용중이라면 etcd에 대한 접근을 제한하는 것이 필요
- 시크릿을 암호화해서 저장하는 것도 가능하지만 그건 쿠버네티스 클러스터를 직접 설치해서 사용할 때 옵션으로 별도로 지정해야 한다.
