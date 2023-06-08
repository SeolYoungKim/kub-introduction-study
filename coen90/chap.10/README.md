# 10 컨피그맵

## 10.1 컨피그맵 사용하기

```yaml
# configmap/config-dev.yaml
apiVersion: apps/v1
kind: ConfigMap
metadata:
  name: config-dev
  namespace: default
data:
  DB_URL: localhost
  DB_USER: myuser
  DB_PASS: mypass
  DEBUG_INFO: debug
```

```yaml
$ kubectl apply -f configmap/config-dev.yaml
configmap/config-dev created
$ kubectl describe configmap config-dev 
Name:         config-dev
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
DB_PASS:
----
mypass
DB_URL:
----
localhost
...
```

- 컨피그맵을 컨테이너에서 불러와서 사용하는 방법은 크게 컨피그맵 설정 중 일부만 불러와서 사용하기, 설정 전체를 한꺼번에 불러와서 사용하기, 볼륨에 불러와서 사용하기 정도가 있다.

## 10.2 컨피그맵 설정 중 일부만 불러와서 사용하기

```yaml
# configmap/deployment-config01.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configapp
  labels: 
    app: configapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: configapp
  template:
    metadata:
      labels:
        app: configapp
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DEBUG_LEVEL # env에 DEGUB_LEVEL=debug 가 들어감
          valueFrom:
            configMapKeyRef:
              name: config-dev
              key: DEBUG_LEVEL
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: configapp
  name: configapp-svc
  namespace: default
spec:
  ports: 
  - nodePort: 30800
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: configapp
  type: NodePort
```

## 10.3 컨피그맵 설정 전체를 한꺼번에 불러와서 사용하기

```yaml
# configmap/deployment-config02.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configapp
  labels: 
    app: configapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: configapp
  template:
    metadata:
      labels:
        app: configapp
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
        envFrom: # env에 config-dev config map 데이터 전체가 들어감
        - configMapRef:
            name: config-dev
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: configapp
  name: configapp-svc
  namespace: default
spec:
  ports: 
  - nodePort: 30800
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: configapp
  type: NodePort
```

## 10.4 컨피그맵을 봄륨에 불러와서 사용하기

```yaml
# configmap/deployment-config03.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configapp
  labels: 
    app: configapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: configapp
  template:
    metadata:
      labels:
        app: configapp
    spec:
      containers:
      - name: testapp
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: config-dev
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: configapp
  name: configapp-svc
  namespace: default
spec:
  ports: 
  - nodePort: 30800
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: configapp
  type: NodePort
```

- kubectl apply 를 하면 config-dev 컨피그맵에 해당하는 파드가 생성된다.

```bash
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS        AGE
configapp-7996464dd7-r5wjd            1/1     Running   0               42s

$ kubectl exec -it configapp-7996464dd7-r5wjd sh

~ # cd /etc/config
/etc/config # ls
DB_PASS     DB_URL      DB_USER     DEBUG_INFO
```

- 컨테이너 안 /etc/config 에 파일이 생성된다.

http://localhost:30800/volume-config?path=/etc/config/DB_USER 로 접속하면 볼 수 있다.
