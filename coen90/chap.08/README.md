# 8 인그레스

## 8.1 인그레스의 개념

- 클러스터 외부에서 안으로 접근하는 요청들을 어떻게 처리할지 정의해둔 규칙 모음이다.
- 인그레스는 규칙들을 정의해둔 자원이고, 실제로 동작시키는 것은 인그레스 컨트롤러이다

```yaml
# ingress/v1beta1
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotaions:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com # 다음 주소로 요청이 들어오면 아래 rule에 따라 처리한다.
    http:
      paths:
      - path: /foo1 
        backend:
          serviceName: s1
          servicePort: 80 # foo.bar.com/foo1 요청은 s1 서비스의 80번포트로 보내라
      - path: /bars2
        backend:
          serviceName: s2
          servicePort: 80 # foo.bar.com/bars2 요청은 s2 서비스의 80번포트로 보내라
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80 # foo.bar.com 요청은 s2 서비스의 80번포트로 보내라
```

- 이처럼 인그레스를 사용하면 클러스터 외부에서 오는 요청을 다양한 방식으로 처리 가능하다

## 8.2 ingress-nginx 컨트롤러

- k8s가 공식적으로 제공하는 인그레스 컨트롤러는 gce용 ingress-gce, nginx용 ingress-nginx
- 여기서는 ingress-nginx를 사용해본다

```bash
$ git clone https://github.com/kubernetes/ingress-nginx.git
```

- 이후 ingress-nginx 네임스페이스를 생성하고 kubectl apply -k 명령어를 실행하면 NodePort 타입 기반의 ingress-nginx 컨트롤러에 접근하는 서비스까지 만들어진다.
- 별도의 인그레스 설정이 없을 때는 에러 메시지가 나타난다.
- 디플로이먼트를 인그레이스에 연결해 인그레이스에 지정했던 이름을 이용한 디플로이먼트 서비스를 생성할 3수 있다.
    ![스크린샷 2023-06-07 오후 10 26 18](https://github.com/Coen90/kub-introduction-study/assets/81370558/383b8e0c-a24b-4336-a6d2-e84ac608c518)
    

## 8.3 인그레스 SSL 설정하기

- 인그레스로 SSL 인증서를 설정하면 파드 각각에 SSL 설정을 따로 할 필요 없다.
    ![스크린샷 2023-06-07 오후 10 27 01](https://github.com/Coen90/kub-introduction-study/assets/81370558/dc2e6156-2704-40c7-bba0-fd8da6c5e5f0)
    

## 8.4 무중단 배포를 할 때 주의할 점

- 인그레이스를 이용해 외부에서 컨테이너에 접근할 수 있다. 이때 새로운 버전의 컨테이너 배포를 할 때는 아래와 같다
    ![스크린샷 2023-06-07 오후 10 29 00](https://github.com/Coen90/kub-introduction-study/assets/81370558/34feea26-74c3-4db7-beb3-5061c8759fcb)

- 새로운 파드(v2)가 생성되고 헬스체크가 성공한 후 v2 쪽으로 트래픽을 보낸다. 이후 v1을 제거한다.

### 8.4.1 maxSurge와 maxUnavailable 필드 설정

- 파드관리를 RollingUpdate로 설정했을 때 .maxSurge와 .maxUnavailable 필드 설정이 필요하다.
- .maxSurge : 디플로이먼트에 설정된 기본 파드 개수 + 여분의 파드 몇개 더 추가할수 있는지 설정
- .maxUnavailable : 디플로이먼트 업데이트동안 몇개의 파드를 이용할 수 없어도 되는지 설정

### 8.4.2 파드가 readinessProbe를 지원하는지 확인

- 무중단 배포에서 신경써서 봐야 할 프로브이다.
- 실제로 컨테이너가 서비스 요청을 처리할 준비가 되었는지 진단한다. readinessProbe가 OK 상태여야 해당 파드와 연결된 서비스에 IP가 추가되고 트래픽을 받을 수 있다.
- readinessProbe가 안될때 대체
    - .spec.minReadySeconds로 최소 대기 시간을 설정해 설정 시간동안 트래픽을 받지 않는다.
    - readinessProbe가 더 우선순위이기에 ok상태가 되면 spec.minReadySeconds를 무시하고 트래픽 보냄

### 8.4.3 쿠버네티스와 컨테이너 안에 그레이스풀 종료 설정

- 그레이스풀 종료 : SIGTERM 신호를 받았을 때 기존에 받은 요청만 처리를 완료하고 새 요청을 받지 않는 것
- 그레이스풀 종료가 설정되지 않으면 아래와 같은 문제가 생긴다.
![스크린샷 2023-06-07 오후 10 36 46](https://github.com/Coen90/kub-introduction-study/assets/81370558/fb561d42-0ca3-4001-83ae-de8e0e0e8318)

- kubelet에서 파드에 SIGTERM 신호를 보낸 후 일정 시간동안 그레이스풀 종료가 되지 않으면 강제로 SIGKILL 신호를 보내 파드를 종료한다. .terminationGracePeriodSeconds 필드로 설정하며 기본 대기 시간은 30초
- 그레이스풀 종료 설정을 못하는 경우 대체
    - 파드 생명 주기 중 훅을 설정해 비슷한 효과를 낸다.
