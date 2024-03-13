## Q1

Deploy a pod named nginx-448839 using the nginx:alpine image.

Once done, click on the Next Question button in the top right corner of this panel. You may navigate back and forth freely between all questions. Once done with all questions, click on End Exam. Your work will be validated at the end and score shown. Good Luck!

```
Name: nginx-448839

Image: nginx:alpine
```

## A1

#### Pod 생성 문제

- `k run [POD NAME] --image=[IMAGE NAME] -o yaml --dry-run=client > [YAML파일명]`

#### 코드

> 해설: kubectl run nginx-448839 --image=nginx:alpine====> 머가다른데

방법 1. 명령어

`kubectl run nginx-448839 --image=nginx:alpine`

방법 2. YAML 파일 생성

`k run nginx-448839 --image=nginx:alpine -o yaml --dry-run=client > nginx-448839.yaml`

`k apply -f nginx-448839.yaml`

#### 회고

- `--dry-run=client` 사용시 `k create`가 아닌 `k run` 사용

## Q2

Create a namespace named apx-z993845

```
Namespace: apx-z993845
```

## A2

#### Namespace 생성 문제

#### 코드

`k create namespace apx-z993845`

## Q3

Create a new Deployment named httpd-frontend with 3 replicas using image httpd:2.4-alpine

```
Name: httpd-frontend

Replicas: 3

Image: httpd:2.4-alpine
```

## A3

#### Deployment 생성 문제

#### 코드

`k create deployment httpd-frontend --image httpd:2.4-alpine --replicas 3`

## Q4

Deploy a messaging pod using the redis:alpine image with the labels set to tier=msg.

```
Pod Name: messaging

Image: redis:alpine

Labels: tier=msg
```

## A4

#### Pod deploy 문제

#### 코드

`k run messaging --image redis:alpine --labels tier=msg -o yaml --dry-run=client > messaging.yaml`

`k apply -f messaging.yaml`

## Q5 -Again

A replicaset rs-d33393 is created. However the pods are not coming up. Identify and fix the issue.

Once fixed, ensure the ReplicaSet has 4 Ready replicas.

```
Replicas: 4
```

## A5

#### Replicaset 생성 문제

#### 코드

ReplicaSet 확인

`k describe replicaset rs-d33393`

- 이때 pod name 조회 가능

Pod 확인

`k describe pod rs-d33393-r6qcr`

- 이때 rs-d33393-r6qcr은 pod name
- `Error: InvalidImageName` 발생

ReplicaSet의 image 정상적으로 변경

- ReplicaSet yaml 파일 생성 후 image 변경

`k get rs rs-d33393 -o yaml > rs-d33393.yaml`

- 기존의 ReplicaSet을 변경한 파일로 변경

`k replace -f rs-d33393.yaml --force`

## Q6

Create a service messaging-service to expose the redis deployment in the marketing namespace within the cluster on port 6379.

Use imperative commands

```
Service: messaging-service

Port: 6379

Use the right type of Service

Use the right labels
```

## A6

#### Service 생성 문제

#### 코드

`kubectl expose deployment redis --port=6379 --name messaging-service --namespace marketing`

## Q7

Update the environment variable on the pod webapp-color to use a green background.

```
Pod Name: webapp-color

Label Name: webapp-color

Env: APP_COLOR=green
```

## A7

#### Environment Variable 문제

#### 코드

`k get pod webapp-color -o yaml > webapp-color.yaml`

- 아래 코드처럼 수정

```
spec:
  containers:
  - env:
    - name: APP_COLOR
      value: green
```

`k replace -f webapp-color.yaml --force`

## Q8

Create a new ConfigMap named cm-3392845. Use the spec given on the below.

```
ConfigName Name: cm-3392845

Data: DB_NAME=SQL3322

Data: DB_HOST=sql322.mycompany.com

Data: DB_PORT=3306
```

## A8

#### ConfigMap 문제

#### 코드

`k create cm -h`로 create 명령어 option 확인

`k create configmap cm-3392845 -o yaml --dry-run=client > cm-3392845.yaml`

`cm-3392845.yaml`

```
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: cm-3392845
data:
  DB_NAME: "SQL3322"
  DB_HOST: "sql322.mycompany.com"
  DB_PORT: "3306"
```

## Q9

Create a new Secret named db-secret-xxdf with the data given (on the below).

```
Secret Name: db-secret-xxdf

Secret 1: DB_Host=sql01

Secret 2: DB_User=root

Secret 3: DB_Password=password123
```

## A9

#### Secret 문제

#### 코드

`k create secret generic -h`로 create 명령어 option 확인

`kubectl create secret generic db-secret-xxdf --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123`

## Q10

Update pod app-sec-kff3345 to run as Root user and with the SYS_TIME capability.

```
Pod Name: app-sec-kff3345

Image Name: ubuntu

SecurityContext: Capability SYS_TIME
```

## A10

#### Pod의 SecurityContext 문제

#### 코드

`k get pod app-sec-kff3345 -o yaml>app-sec-kff3345.yaml`

`app-sec-kff3345.yaml`

```
spec:
  containers:
    ...
    securityContext:    #spec.containers.securityContext
        capabilities:
        add: ["SYS_TIME"]
```

`k replace -f app-sec-kff3345.yaml --force`

## Q11 -Again

Export the logs of the e-com-1123 pod to the file /opt/outputs/e-com-1123.logs

It is in a different namespace. Identify the namespace first.

```
Task Completed
```

## A11

#### Log 문제

#### 코드

`k get pods --all-namespaces`로 namespace 확인

`k describe pod e-com-1123 -n e-commerce`에서 container name (simple-webapp) 확인

## Q12

Create a Persistent Volume with the given specification.

```
Volume Name: pv-analytics

Storage: 100Mi

Access modes: ReadWriteMany

Host Path: /pv/data-analytics
```

## A12

#### Volume 문제

#### 코드

`pv-analytics.yaml`

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/pv/data-analytics"
```

## Q13 -Again

Create a redis deployment using the image redis:alpine with 1 replica and label app=redis. Expose it via a ClusterIP service called redis on port 6379. Create a new Ingress Type NetworkPolicy called redis-access which allows only the pods with label access=redis to access the deployment.

```
Image: redis:alpine

Deployment created correctly?

Service created correctly?

Network Policy allows the correct pods?

Network Policy applied on the correct pods?
```

## A13

#### Network Policy 문제

#### 코드

Deployment 생성

`k create deploy redis --image redis:alpine --replicas 1`

`k get deploy redis -o yaml > redis.yaml`

- label 일치(이미 일치)

ClusterIP Service 생성

`k expose deployment redis --port 6379 --name redis`

Network Policy 생성

`redis-access.yaml`

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-access
  namespace: default
spec:
  podSelector:
    matchLabels:
      access: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: redis
    ports:
    - protocol: TCP
      port: 6379
```

## Q14 -Again

Create a Pod called sega with two containers:

Container 1: Name tails with image busybox and command: sleep 3600.
Container 2: Name sonic with image nginx and Environment variable: NGINX_PORT with the value 8080.

```
Container Sonic has the correct ENV name

Container Sonic has the correct ENV value

Container tails created correctly?
```

## A14

#### 하나의 Pod에 다중 Container

#### 코드

`sega.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: sega
spec:
  containers:
  - image: busybox
    name: tails
    command:
    - sleep
    - 3600
  - image: nginx
    name: sonic
    env:
    - name: NGINX_PORT
      value: "8080"
```