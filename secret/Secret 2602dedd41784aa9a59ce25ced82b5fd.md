# Secret

대부분 configmap의 사용방식과 같고 

다면 민감한 정보를 저장하기 위해 좀 더 세분화된 사용 방법을 제공

# 사용방법 익히기

```yaml
kubectl create generic my-password --from-literal password=1q2w3e4r
```

### 환경변수로 가져오기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-example
spec:
  containers:
    - name: my-container
      image: busybox
      args: ['tail', '-f', '/dev/null']
      envFrom:
      - secretRef:
          name: my-password
```

### 환경변수에서 선택해서 가져오기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selective-secret-env-example
spec:
  containers:
  - name: my-container
    image: busybox
    args: ['tail', '-f', '/dev/null']
    env:
    - name: YOUR_PASSWORD
      valueFrom:
        secretKeyRef:
          name: our-password
          key: pw2
```

### 

### 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: my-container
      image: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: secret-volume          #  volumes에서 정의한 시크릿 볼륨 이름
        mountPath: /etc/secret             # 시크릿의 데이터가 위치할 경로
  volumes:
  - name: secret-volume            # 시크릿 볼륨 이름
    secret:
      secretName: our-password                  # 키-값 쌍을 가져올 컨피그맵 이름
```

### 선택해서 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selective-volume-pod
spec:
  containers:
  - name: my-container
    image: busybox
    args: [ "tail", "-f", "/dev/null" ]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
  volumes:
  - name: secret-volume
    secret:
      secretName: our-password          # our-password 라는 시크릿을 사용
      items:                       # 시크릿에서 가져올 키-값의 목록
      - key: pw1                    # pw1라는 키에 대응하는 값이 사용됨.
        path: password1           # 최종 파일은 /etc/config/password1이 됨찯
```

## Docker registry 타입 시크릿

```yaml
kubectl crate secret docker-registry registry-auth-by-cmd \
--docker-username=${DOCKER_USERNAME}
--docker-password=${DOCKER_PASSWORD}
```

## TLS타입 시크릿

```yaml
openssl req -new -newkey  rsa:4096 -days 365 -nodes -x509 -subj "/CN=example.io" -keyout cert.key -out cert.crt
```

```yaml
kubectl create secret tls my-tls-secret --cert cert.crt --key cert.key
```