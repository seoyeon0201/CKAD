## Q1

We have deployed a few pods in this cluster in various namespaces. Inspect them and identify the pod which is not in a Ready state. Troubleshoot and fix the issue.

Next, add a check to restart the container on the same pod if the command ls /var/www/html/file_check fails. This check should start after a delay of 10 seconds and run every 60 seconds.


You may delete and recreate the object. Ignore the warnings from the probe.

## A1

#### `Readiness`와 `Liveness` 문제

- `container port`와 `readinessProbe port`가 일치해야함
- `livenessProbe.initialDelaySeconds`와 `livenessProbe.periodSeconds` 설정
- `livenessProbe`를 통해 특정 폴더의 성공 유무에 따라 container 처리

#### 코드

`nginx1401.yaml`
```
spec:
  containers:
  - image: kodekloud/nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    ports:
    - containerPort: 9080   #1-1. container port와 readinessProbe port 일치
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /
        port: 9080        #1-2. container port와 readinessProbe port 일치
        scheme: HTTP
      successThreshold: 1
      timeoutSeconds: 1
    livenessProbe:        #3. livenessProbe로 아래 command의 성공 유무에 따라 container 처리 
      exec:
        command:
        - ls
        - /var/www/html/file_check
      initialDelaySeconds: 10 #2-1. initialDelaySeconds로 delay 설정 
      periodSeconds: 60       #2-2. periodSecond로 주기 설정
```

#### 회고

- livenessProbe를 사용해 성공 유무에 따라 container를 처리할 수 있음
- `k replace -f [YAML파일명] --force`로 기존의 pod 변경

## Q2

Create a cronjob called dice that runs every one minute. Use the Pod template located at /root/throw-a-dice. The image throw-dice randomly returns a value between 1 and 6. The result of 6 is considered success and all others are failure.

The job should be non-parallel and complete the task once. Use a backoffLimit of 25.

If the task is not completed within 20 seconds the job should fail and pods should be terminated.

## A2

#### `CronJob` 문제

- /root/throw-a-dice에 위치한 `pod image를 cronjob 이미지에 동일하게 적용`
- 1분에 한 번씩 실행되므로 spec.schedule이 `*/1 * * * *`
- jobTemplate은 backofLimit이 주어진 25로 설정
- jobTemplate에서 activeDeadlineSeconds를 20초로 설정해 20초가 지난 후에는 pod가 종료되도록 설정
- jobTemplate에서 completions를 1로 설정
- restartPolicy는 Never로 설정

#### 코드

`cronjob.yaml`
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dice
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      completions: 1  #기본 completions가 1이라 없어도 됨 
      activeDeadlineSeconds: 20
      backoffLimit: 25
      template:
        spec:
          containers:
          - name: dice
            image: throw-dice
            imagePullPolicy: IfNotPresent
          restartPolicy: Never
```

#### 회고
- pod의 위치를 알려준 것은 pod의 정보(ex.image)를 알려준 것 => cronjob 작성시 필요한 container의 스펙을 알려줌
- sepc.jobTemplate.spec.completions는 문제에 주어지지만 기본적으로 1
- cronjob 주기 설정 시 `*/2 * * * *`와 `2 * * * *`는 다름
  - 전자는 2분마다 반복됨을 의미하고 후자는 매시간 2분에 해당 container를 실행하는 것을 의미

## Q3

Create a pod called my-busybox in the dev2406 namespace using the busybox image. The container should be called secret and should sleep for 3600 seconds.

The container should mount a read-only secret volume called secret-volume at the path /etc/secret-volume. The secret being mounted has already been created for you and is called dotfile-secret.

Make sure that the pod is scheduled on controlplane and no other node in the cluster.

## A3

#### secret 문제

- dev2406이라는 namespace 있는지 확인 
  - 없는 경우 namespace 먼저 생성
  - 있는 경우 pod의 namespace dev2406으로 설정
- `nodeName`을 지정해 pod가 controlplane에 존재하도록 함
- image를 busybox로 설정
- `command`에 3600만큼 sleep되도록 함
- `secretName`이 dotfile-secret인 secret을 volume으로 가져옴
  - dotfile-secret은 이전에 이미 만들어져있음
- volumeMounts로 이전에 가져온 secret을 가져오되 readOnly 상태로 가져오고 mountPath는 /etc/secret-volume 위치에 저장

#### 코드

`my-busybox.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: my-busybox
  namespace: dev2406
spec:
  nodeName: controlplane
  containers:
  - name: secret
    image: busybox
    ports:
    - containerPort: 80
    command: 
    - sleep 
    - "3600"
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: /etc/secret-volume
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
```

#### 회고
- 이미 만들어진 secret을 secretName으로 가져와 사용
- `spec.nodeName` 대신 아래 코드도 가능. 동일한 기능
```
  nodeSelector:
    kubernetes.io/hostname: controlplane
```

## Q4

Create a single ingress resource called ingress-vh-routing. The resource should route HTTP traffic to multiple hostnames as specified below:

The service video-service should be accessible on http://watch.ecom-store.com:30093/video

The service apparels-service should be accessible on http://apparels.ecom-store.com:30093/wear


Here 30093 is the port used by the Ingress Controller

## A4

#### Ingress 문제

- host name에 따라 실행하는 서비스가 다르게 설정

#### 코드

`ingress.yaml`
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-vh-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: watch.ecom-store.com
    http:
      paths:
      - pathType: Prefix
        path: "/video"
        backend:
          service:
            name: video-service
            port:
              number: 8080
  - host: apparels.ecom-store.com
    http:
      paths:
      - pathType: Prefix
        path: "/wear"
        backend:
          service:
            name: apparels-service
            port:
              number: 8080
```


#### 회고

- HTTP 경로가 http://watch.ecom-store.com:30093/video라 port 번호가 30093인줄 알았는데, service의 port 번호를 적어야 함
  - `kubectl get svc` 사용
- 아래 코드 없으면 틀림
  - `rewrite-target`은 ingress에 정의된 경로로 들어오는 요청을 설정된 경로로 전달함을 의미
  - 서비스를 구분하는 path는 그대로두되, 하위 path만을 요청하고 싶은 경우 Rewrite annotation 사용
    -`원하는 service에` 하위 path를 그대로 요청 가능
    - Ingress로 /something을 요청하면 실제 서비스는 /로 요청

```
annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

## Q5

A pod called dev-pod-dind-878516 has been deployed in the default namespace. Inspect the logs for the container called log-x and redirect the warnings to /opt/dind-878516_logs.txt on the controlplane node

## A5

#### log 문제

- container 이름이 log-x인 container 로그 확인
  - `kubectl logs [POD이름] -c [CONTAINER이름]`
- 위 container에 존재하는 모든 WARNING을 `/opt/dind-878516_logs.txt`에 리다이렉트
  - `kubectl logs [POD이름] -c [CONTAINER이름] | grep WARNING > [옮길 경로]`
  - kubectl logs dev-pod-dind-878516 -c log-x | grep WARNING > /opt/dind-878516_logs.txt
- cat 명령어로 확인
  - `cat [조회하고싶은위치]`
  - cat /opt/dind-878516_logs.txt

#### 코드

`kubectl logs dev-pod-dind-878516 -c log-x | grep WARNING > /opt/dind-878516_logs.txt`

#### 회고

- log 중 WARNING을 `grep할 때에는 반드시 대문자`로 써야 가져올 수 있음
