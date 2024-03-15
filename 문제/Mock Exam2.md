## Q1

Create a deployment called my-webapp with image: nginx, label tier:frontend and 2 replicas. Expose the deployment as a NodePort service with name front-end-service , port: 80 and NodePort: 30083

```
Deployment my-webapp created?

image: nginx

Replicas = 2 ?

service front-end-service created?

service Type created correctly?

Correct node Port used?
```

## A1

#### NodePort Service 문제

#### 코드

Deployment 생성 후 Service expose

- service yaml 파일 생성
  - command에 nodePort port번호를 지정하는 옵션이 없음

`k expose deployment my-webapp --port 80 --name front-end-service --type=NodePort -o yaml --dry-run=client > front-end-service.yaml`

- yaml 파일에 nodePort port 변경

확인 

- 우측 상단의 "View Port" 클릭해 nodePort 번호 입력해 연결 확인
- `curl [CLUSTER IP]:[PORT]`로 확인
  - 이떄 port는 nodeport가 아닌 service port

### 회고

- expose 명령어로 서비스 노출시킬 때 --port 옵션은 service port 번호이고, node port 번호는 yaml 파일에 직접 작성해야함 !

## Q2

Add a taint to the node node01 of the cluster. Use the specification below:

key: app_type, value: alpha and effect: NoSchedule

Create a pod called alpha, image: redis with toleration to node01.

```
node01 with the correct taint?

Pod alpha has the correct toleration?
```

## A2

#### Taints and Tolerations 문제

#### 코드

taint 생성

`kubectl taint nodes node01 app_type=alpha:NoSchedule`

pod 생성

`alpha.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: alpha
  name: alpha
spec:
  containers:
  - image: redis
    name: alpha
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  tolerations:
  - key: "app_type"
    operator: "Equal"
    value: "alpha"
    effect: "NoSchedule"
status: {}
```

- `k describe pod alpha` 또는 `k get pod -o wide`에서 node 확인 가능

### 회고

- spec.tolerations.operator의 기본 값이 "Equal"이므로 effect, key, value만 작성해도 됨

## Q3 ✏️

Apply a label app_type=beta to node controlplane. Create a new deployment called beta-apps with image: nginx and replicas: 3. Set Node Affinity to the deployment to place the PODs on controlplane only.

NodeAffinity: requiredDuringSchedulingIgnoredDuringExecution

```
controlplane has the correct labels?

Deployment beta-apps: NodeAffinity set to requiredDuringSchedulingIgnoredDuringExecution ?

Deployment beta-apps has correct Key for NodeAffinity?

Deployment beta-apps has correct Value for NodeAffinity?

Deployment beta-apps has pods running only on controlplane?

Deployment beta-apps has 3 pods running?
```

## A3

#### Node Label 및 Node Affinity 문제

#### 코드

`kubectl label node controlplane app_type=beta`로 node에 label 적용

`kubectl get nodes --show-labels`로 node에 붙은 label 조회 가능

`beta-apps.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: beta-apps
  name: beta-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: beta-apps
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: beta-apps
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: app_type
                operator: In
                values:   #또는 values: ["beta"]
                - beta
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

`k get pod -o wide`로 확인

## Q4 ✏️

Create a new Ingress Resource for the service my-video-service to be made available at the URL: http://ckad-mock-exam-solution.com:30093/video.

To create an ingress resource, the following details are: -

```
annotation: nginx.ingress.kubernetes.io/rewrite-target: /

host: ckad-mock-exam-solution.com

path: /video

Once set up, the curl test of the URL from the nodes should be successful: HTTP 200

http://ckad-mock-exam-solution.com:30093/video accessible?
```

## A4

#### Ingress 문제

#### 코드

방법1

`k create ingress ingress --rule=ckad-mock-exam-solution.com/video=my-video-service:30093 -o yaml --dry-run=client > ingress.yaml`

`ingress.yaml`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: ckad-mock-exam-solution.com
    http:
      paths:
      - backend:
          service:
            name: my-video-service
            port:
              number: 8080
        path: /video
        pathType: Exact
status:
  loadBalancer: {}
```

방법2

`kubectl create ingress ingress --rule="ckad-mock-exam-solution.com/video*=my-video-service:8080" -o yaml --dry-run=client > ingress.yaml` 

yaml 파일에서 `annotation: nginx.ingress.kubernetes.io/rewrite-target: /` 추가

```
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

- --rule="ckad-mock-exam-solution.com/video*=my-video-service:8080"에서 `/video*`는 video로 시작하는 `모든 path` 의미
  - *(별표) 생략가능 !

`curl http://ckad-mock-exam-solution.com:30093/video`로 연결 확인

## Q5 ✏️

We have deployed a new pod called pod-with-rprobe. This Pod has an initial delay before it is Ready. Update the newly created pod pod-with-rprobe with a readinessProbe using the given spec


httpGet path: /ready

httpGet port: 8080
```
readinessProbe with the correct httpGet path?

readinessProbe with the correct httpGet port?
```
## A5

#### ReadinessProbe 문제

#### 코드

readinessProbe 코드 추가

`rprobe.yaml`
```
spec.
  containers:
    ...
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
```

## Q6

Create a new pod called nginx1401 in the default namespace with the image nginx. Add a livenessProbe to the container to restart it if the command ls /var/www/html/probe fails. This check should start after a delay of 10 seconds and run every 60 seconds.


You may delete and recreate the object. Ignore the warnings from the probe.
```
Pod created correctly with the livenessProbe?
```

## A6

#### LivenessProbe 문제

#### 코드

`nginx1401.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx1401
  name: nginx1401
spec:
  containers:
  - image: nginx
    name: nginx1401
    resources: {}
    livenessProbe:
      exec:   #또는 exec.command: ["ls /var/www/html/probe"]
        command:
        - ls
        - /var/www/html/probe
      initialDelaySeconds: 10
      periodSeconds: 60
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

## Q7

Create a job called whalesay with image docker/whalesay and command "cowsay I am going to ace CKAD!".

completions: 10

backoffLimit: 6

restartPolicy: Never


This simple job runs the popular cowsay game that was modifed by docker…
```
Job "whalesay" uses correct image?

Job "whalesay" configured with completions = 10?

Job "whalesay" with backoffLimit = 6

Job run's the command "cowsay I am going to ace CKAD!"?

Job "whalesay" completed successfully?
```
## A7

#### Job 문제
#### 코드

`job.yaml`
```
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: whalesay
spec:
  completions: 10
  backoffLimit: 6
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - image: docker/whalesay
        name: whalesay
        resources: {}
        # command 방법1
        command:
        - sh
        - -c
        - "cowsay I am going to ace CKAD!"

        # command 방법2

        command: ["sh","-c", "cowsay I am going to ace CKAD!"]

        # command 방법3
        command: ["printenv"]
        args: ["cowsay I am going to ace CKAD!"]

      restartPolicy: Never
status: {}
```

## Q8

Create a pod called multi-pod with two containers.

Container 1:
name: jupiter, image: nginx

Container 2:
name: europa, image: busybox
command: sleep 4800

Environment Variables:

Container 1:

type: planet

Container 2:

type: moon

```
Pod Name: multi-pod

Container 1: jupiter

Container 2: europa

Container europa commands set correctly?

Container 1 Environment Value Set

Container 2 Environment Value Set
```

## A8

#### Multi container 문제
#### 코드

`multi-pod.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-pod
  name: multi-pod
spec:
  containers:
  - image: nginx
    name: jupiter
    env:
    - name: type
      value: "planet"
    resources: {}
  - image: busybox
    name: europa
    # command 방법1
    command: ["/bin/sh", "-c","sleep 4800"]

    # command 방법2
    command: ["sleep", "4800"]

    # command 방법3
    command:
    - sleep
    - "4800"
    env:
    - name: type
      value: "moon"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

## Q9

Create a PersistentVolume called custom-volume with size: 50MiB reclaim policy:retain, Access Modes: ReadWriteMany and hostPath: /opt/data

```
PV custom-volume created?

custom-volume uses the correct access mode?

PV custom-volume has the correct storage capacity?

PV custom-volume has the correct host path?
```

## A9
#### Persistent Volume 문제
#### 코드

`custom-volume.yaml`
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: custom-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 50Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/opt/data"
```

## 궁금증

#### Q1

- nodeport의 nodeport 번호는 command로 지정할 수 없나?
  - 불가능

- NodePort 서비스 생성 후 연결되었는지 확인하는 방법 => 현재는 오른쪽 상단의 View Port로 확인

  - `curl [CLUSTER IP ADDRESS]:[SERVICE PORT]`
    - 이때 CLUSTER IP ADDRESS는 `k get svc`에서 확인 가능
    - SERVICE PORT는 Nodeport가 아닌 port 번호

#### Q9

- PV를 명령어로 생성하는법
  - 불가능