# ReplicaSet

# Replicaset을 사용하는 이유

k8s의 기본 단위인 pod는 여러개의 컨테이너를 추상화해 하나의 애플리케이션으로 동작하도록 만드는 훌륭한 컨테이너 묶음입니다. 

하지만 yaml에 pod만 정의해 생성하게 되면 이 pod의 생애주기는 어떻게 될까요?

예를 들어 앞서 생성했던 2개의 컨테이너가 담겨있는 nginx pod는 다음 두가지 방식으로 삭제할 수 있습니다.

```bash
kubectl delete  -f k8s_yaml/nginx-pod-with-ubuntu.yaml
```

pod를 삭제하면 그 pod의 컨테이너 또한 삭제된 뒤 쿠버네티스에서 영원히 사라지게 됩니다.

이처럼 yaml파일에 pod만 정의해 생성할 경우 해당 pod는 오직 쿠버네티스 유저에 의해 관리됩니다.

단순히 테스트하고 이런 간단한 용도로는 이렇게 사용할 수 있을지 모릅니다. 

하지만 실제로 트레픽을 처리해야 하는 MSA구조의 pod라면 이런 방식을 사용하기는 어렵습니다. 

여러 개의 동을한 컨테이너를 생성한 뒤 트레픽을 적절히 분배될 수 있어야 합니다. 

가장 단순하게 생각할 수 있는 방식은 power 수동 세팅입니다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-a
spec:
  containers:
  - name: my-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP

---
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-b
spec:
  containers:
  - name: my-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
```

이런 방식은 적절하지 않음 

1. pod가 많아질수록 일일히 정의하는 것은 비효율적 
2. pod가 어떤 이유로 삭제되거나 장애가 발생해 pod에 더이상 접근하지 못하게 됐을 때, 직접 pod를 지우고 새로 올리지 않는 한 다시 복구되지 않음

⇒ 이러한 문제를 해결해주는것이 replicaset

replicaset이 수행하는 역할

- 정해진 수의 동일한 pod가 항상 실행되도록 관리
- 노드 장애 등의 이유로 pod를 사용할 수 없다면 다른 node에서 pod를 다시 생성

따라서 동일한 nginx pod를 안정적으로 여러 개 실행할 수도 있고 워커 노드에 장애가 생기더라도 정해진 개수의 pod 유지 가능 

# Replicaset 사용하기

```bash
vi k8s_yaml/replicaset-nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels: 
        app: my-nginx-pods-label
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

- spec.replicas : replica를 몇개 유지할 것인지 정합니다.
- spec.template : pod를 생성할 때 사용될 탬플릿입니다. 
template 아래 내용은 아까 pod를 정의한 내용과 같습니다. 즉. pod를 사용했던 내용을 동일하게 replicaset에 정의해서 replicaset이 어떻게 pod를 만들게 할까를 정의하는 것입니다.

replicaset 실행

```bash
kubectl apply -f k8s_yaml/replicaset-nginx.yaml
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled.png)

pod 확인 

```bash
kubectl get po
```

replicaset 확인

```bash
kubectl get rs
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%201.png)

pod 하나를 지워보겠습니다. 

```bash
kubectl delete pod replicaset-nginx-69qlw
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%202.png)

새로운 pod가 바로 만들어지는 것을 알 수 있습니다.

자 이제 replica 수를 늘려보겠습니다. 

yaml 파일에 replicas 를 4로 변경합니다. 

```bash
vi k8s_yaml/replicaset-nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 4
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels: 
        app: my-nginx-pods-label
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%203.png)

이제 다시 적용해보겠습니다.

```bash
kubectl apply -f k8s_yaml/replicaset-nginx.yaml
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%204.png)

configured 라고 뜹니다. 

새로운 리소스가 만들어진 것이 아닌 기존 리소스의 정보가 변경된 것이기 때문입니다.

# Replicaset의 동작원리

마치 replicaset은 pod와 모종의 무언가로 연결된 거 같은 느낌을 받습니다. 

replicaset을 생성하면 pod가 생성되고 삭제하면 pod도 삭제되니 말입니다. 

하지만 실제로는 연결되어 있지 않습니다.

오히려 느슨한 연결(loosely coupled)을 유지하고 있으며, 이러한 느슨한 연결은 pod와 replicaset의 정의 중 라벨셀렉터를 이용해 이뤄집니다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 4
  selector:
    **matchLabels:
      app: my-nginx-pods-label**
  template:
    metadata:
      name: my-nginx-pod
      **labels: 
        app: my-nginx-pods-label**
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

이전에 pod를 생성할 때 metadata 항목에서는 리소스의 부가적인 정보를 설정할 수 있다고 설명했습니다. 

metadata에는 리소스의 이름뿐만 아니라 annotation, label도 포함됩니다. 

특히 label은 pod 등의 k8s 리소스를 분류할 때 유용하게 사용할 수 있는 메타데이터

label은 k8s리소스의 부가적인 정보를 표현할 수 있을 뿐만 아니라, 서로 다른 오브젝트가 서로를 찾아야 할 때 사용되기도 합니다. 

예를 들어 replicaset은 spec.selector.matchLabel에 정의된 label을 통해 생성해야 하는 pod를 찾습니다.

즉 app : my-nginx-pods-label 라벨을 가지는 pod의 개수가 replicas 항목에 정의된 숫자인 3개와 일치하지 않으면 pod를 정의하는 pod template 항목의 내용으로 pod를 생성

replicaset을 처음 생성했을 때는 app:my-nginx-pods-label 라벨을 가지는 pod가 존재하지 않기 때문에 template에 정의된 pod 설정으로 3개의 pod를 생성

그렇다면 미리 해당 라벨을 가지는 pod를 만들어놓은 후 replicaset을 실행시켜보겠습니다. 

```bash
vi k8s_yaml/nginx-pod-without-rs.yaml
```

```bash
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
  labels:
    app: my-nginx-pods-label
spec:
  containers:
  - name: my-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f k8s_yaml/nginx-pod-without-rs.yaml
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%205.png)

확인해보겠습니다. 

```bash
kubectl get pods --show-labels
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%206.png)

```bash
kubectl get pods  -l app
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%207.png)

```bash
kubectl get pods -l app=my-nginx-pods-label
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%208.png)

이 상태로 다시 레플리카셋 생성해보겠습니다. 

```bash
kubectl apply -f k8s_yaml/replicaset-nginx.yaml
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%209.png)

```bash
kubectl get po
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2010.png)

replicaset이 붙은게 3개만 만들어졌습니다. 

이제 수동으로 생성한 pod를 삭제해보겠습니다. 

```bash
kubectl delete pods my-nginx-pod
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2011.png)

바로 추가로 하나가 생깁니다.

이번에는 라벨을 바꿔보겠습니다. 

```bash
kubectl edit pods replicaset-nginx-9fdsz
```

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Pod
metadata:
  annotations:
    seccomp.security.alpha.kubernetes.io/pod: runtime/default
  creationTimestamp: "2023-04-18T21:09:33Z"
  generateName: replicaset-nginx-
  **~~labels:
    app: my-nginx-pods-label~~**
  name: replicaset-nginx-9fdsz
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: replicaset-nginx
    uid: cabdee0a-0012-41a9-89c2-2c5f3f644031
  resourceVersion: "717451"
  uid: b32c2248-54cb-42a3-992b-aded8135b949
spec:
  containers:
  - image: nginx:latest
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      limits:
        cpu: 500m
        ephemeral-storage: 1Gi
        memory: 2Gi
      requests:
        cpu: 500m
        ephemeral-storage: 1Gi
        memory: 2Gi
    securityContext:
      capabilities:
        drop:
        - NET_RAW
    terminationMessagePath: /dev/termination-log
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2012.png)

```yaml
kubectl get po
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2013.png)

pod 하나가 바로 추가되었습니다. 

그렇다면 여기서 만약 같은 라벨을 가지는 pod를 수동으로 추가한다면 어떻게 될까요?

```bash
kubectl apply -f k8s_yaml/nginx-pod-without-rs.yaml
```

```bash
kubectl get po
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2014.png)

바로 얄짤없이 제거됩니다.

그렇다면 라벨이 없는 pod를 띄운 후 라벨을 추가하면 어떻게 될까요?

```bash
kubectl apply -f k8s_yaml/nginx-pod.yaml
```

```bash
kubectl get po
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2015.png)

라벨을 추가시켜보겠습니다. 

```bash
kubectl edit pods my-nginx-pod
```

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Pod
metadata:
  annotations:
    autopilot.gke.io/resource-adjustment: '{"input":{"containers":[{"name":"my-nginx-container"}]},"output":{"containers":[{"limits":{"cpu":"500m","ephemeral-storage":"1Gi","memory":"2Gi"},"requests":{"cpu":"500m","ephemeral-storage":"1Gi","memory":"2Gi"},"name":"my-nginx-container"}]},"modified":true}'
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"my-nginx-pod","namespace":"default"},"spec":{"containers":[{"image":"nginx:latest","name":"my-nginx-container","ports":[{"containerPort":80,"protocol":"TCP"}]}]}}
    seccomp.security.alpha.kubernetes.io/pod: runtime/default
  creationTimestamp: "2023-04-18T21:18:12Z"
  **labels:
    app: my-nginx-pods-label**
  name: my-nginx-pod
  namespace: default
  resourceVersion: "725204"
  uid: 7a2743c0-1e0c-441a-9a00-722c9d8038a5
spec:
  containers:
  - image: nginx:latest
    imagePullPolicy: Always
    name: my-nginx-container
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      limits:
        cpu: 500m
        ephemeral-storage: 1Gi
        memory: 2Gi
      requests:
        cpu: 500m
        ephemeral-storage: 1Gi
        memory: 2Gi
    securityContext:
      capabilities:
        drop:
        - NET_RAW
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-pp2hs
      readOnly: true
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2016.png)

```bash
kubectl get po
```

![Untitled](ReplicaSet%20732131cc3afb499cb8b09a7b5fc2ca05/Untitled%2017.png)

흔적도 없이 사라졌습니다. 

```bash
kubectl delete replicaset replicaset-nginx
kubectl delete pod replicaset-nginx-9fdsz
```

<aside>
💡 여기서 한가지 오해하지 말아야 할 점은 replicaset의 목적이 pod를 생성하는 것이 아닌 **일정 개수의 pod를 유저하는 것**이라는 점입니다. 현재 pod의 개수가 replicas보다 적으면 pod를 더 생성해 동일하게 유지 하지만 너무 많으면 역으로 pod를 삭제해 relicas 만큼 동일하게 유지

</aside>

# replicaset vs replication controller

이전 버전의 k8s에서는 replicaset이 아닌 replication controller라는 오브젝트를 통해서 pdo의 개수를 유지했습니다. 그러나 k8s의 버전이 올라감에 따라 replication controller는 더이상 사용되지 않으며(deprecated) 그 대신 replicaset이 사용되고 있습니다.

replicaset이 replication controller와는 다른 점은 표현식(matchExpression) 기반의 label selector

예를 들어 다음과 같이 표현 가능 

```bash
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: our-nginx-pods-label
    matchExpressions:
      - key: app2
        values:
          - my-nginx-pods-label
          - your-nginx-pods-label
        operator: In
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: our-nginx-pods-label
        app2: my-nginx-pods-label
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

위 예시는 key가 app인 라벨을 가지고 있는 pod 중에서 values 항목에 정의된 값들이 존재(In)하는 pod들을 대상으로 하겠다는 의미입니다. 

app: our-nginx-pods-label이라는 label을 가지는 pod뿐만 아니라 

app2: my-nginx-pods-label이란느 label을 가지는 pod 또한 관리