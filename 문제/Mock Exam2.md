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

- yaml 파일에 nodePort port번호 작성

우측 상단의 "View Port" 클릭해 nodePort 번호 입력해 연결 확인

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

- `k describe pod alpha`에서 node 확인 가능

### 개념 or 회고

## Q3 - 틀림 다시 !

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
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app_type
                operator: In
                values:
                - "beta"
            topologyKey: topology.kubernetes.io/zone
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

## Q4

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
              number: 30093
        path: /video
        pathType: Exact
status:
  loadBalancer: {}
```

`curl http://ckad-mock-exam-solution.com:30093/video`로 연결 확인

## Q5 - 모르겠음 ㅠ

We have deployed a new pod called pod-with-rprobe. This Pod has an initial delay before it is Ready. Update the newly created pod pod-with-rprobe with a readinessProbe using the given spec


httpGet path: /ready

httpGet port: 8080
```
readinessProbe with the correct httpGet path?

readinessProbe with the correct httpGet port?
```
## A5

#### ReadinessProbe 문제? LivenessProbe 문제?

#### 코드

livenessProbe 코드 추가

`rprobe.yaml`
```
spec.
  containers:
    ...
    livenessProbe:
      httpGet:
        path: /ready
        port: 8080
```
- pod에 오류 발생






## 궁금증

#### Q1

- nodeport의 nodeport 번호는 command로 지정할 수 없나?

- nodePort 서비스 생성 후 연결되었는지 확인하는 방법 => 현재는 오른쪽 상단의 View Port로 확인
