## Labels, Selector and Annotations

1. Labels and Selector

- Labels와 Selector는 그룹으로 묶는 표준 방법
- Labels는 물건마다 붙어있는 속성 
    - Ex. 한 동물에 대해 Class:Mammal, Kind:Domestic, Color:Green 등
- Selectors는 위 항목을 필터링하는 것을 도움
    - Ex. Class==Mammal && Color=Green 등

2. Labels and Selector in Kubernetes

- Kubernetes에는 다양한 유형의 Object 존재
    - Ex. Pod, Service, ReplicaSet, Deployment
- 위 객체를 필터링하고 볼 수 있는 방법 필요 => Type별로, application별로, 혹은 기능별로 
- Label과 Selector를 이용해 개체를 그룹화하고 선택할 수 있음

- `Labels`: 각 Object는 필요에 따라 Label이 붙음 Ex. app, function
    - pod definition file에 metadata>labels에 key-value 포맷으로 추가


- `Selectors`: 선택하는 동안 특정 개체를 필터링할 조건 명시

3. 사용 예시

- 1. Labels
    `pod-definition.yml`
    ```
    apiVersion: v1
    kind: Pod
    metadata:
        name: simple-webapp
        labels: #label 지정
            app: App1
            function: Front-end
    spec:
        containers:
            - name: simple-webapp
              image: simple-webapp
              ports:
                - containerPort: 8080
    ```
- 2. Selectors

`kubectl get pods --selector app=App1`

- 3. ReplicaSet에서의 Labels과 Selector

    - Kubernetes object는 내부에서 label과 selector를 이용해 서로 다른 object 연결
    - `annotation`은 정보 수집 목적으로 다른 세부 사항을 기록하는데 사용

`replicaset-definition.yaml`
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: simple-wepapp
    labels: #replicaSet 자체의 label
        app: App1
        function: Front-end
    #annotations
    annotations:
        buildversion: 1.34
spec:
    replicas: 3
    selector:   #selector 아래의 label과 일치하는 pod 찾기
        matchLabels:
            app: App1   
    template:   #template 섹션 아래의 label은 pod에서 구성된 label
        metadata:
            labels:
                app: App1
                function: Front-end
        spec:
            containers:
                - name: simple-webapp
                  image: simple-webapp
```

- 4. Service에서의 Labels과 Selector

    - service의 selector를 통해 service를 진행할 pod 찾음

`service-definition.yaml`
```
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        app: App1
    ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
```

## Rolling Updates & Rollbacks in Deployments

1. Rollout and Versioning(버전관리)

- 처음 배포 시 Rollout 촉발: 처음 배포시 새로운 deployment revision 생성(Revision1)>app이 upgrade되면, 즉 container version이 새 version에 update되면 새 Rollout이 트리거되고 새 deployment revision 생성(Revision2)

- deployment에 일어난 변화를 추적할 수 있게 해주고 필요하다면 배포 이전 버전으로 되돌릴 수 있게 해줌

2. application을 업그레이드하는 방법

3. Rollout Command

- `k rollout status deployment/[DEPLOYMENT NAME]` : 출시 상태를 실행해서 출시 확인 가능

- `k rollout history deployment/[DEPLOYMENT NAME]` : deployment 역사 확인

4. Deployment Strategy

- 배포된 web application instance replicas가 5개인 경우 가정
- 1. Recreate
    - 기존의 5개의 instance 모두 파괴 후 application instance의 새로운 버전의 instance 5개 배포
    - 단점: 구 버전이 다운되고 새 버전이 업데이트되기 전의 기간동안 application 다운되고 사용자의 접근 불가능
    - 방식: Replicaset이 0으로 축소되었다가 5로 확대

- 2. Rolling Update
    - 기본 배포 전략. 배포 전략 지정하지 않으면 해당 방식 추정
    - 한 번에 다 없애지 않고 구버전을 다운하고 하나씩 새 버전을 올리는 방식
    - 장점: application이 다운되지 않고 업그레이드도 원활
    - 방식: 기존 Replicaset을 하나씩 축소하고 새 ReplicaSet도 하나씩 축소

- Deployment를 업데이트하는 법 => (1) 파일 수정 후 `k apply -f [FILE NAME]` (2)`k set image deployment/[DEPLOYMENT NAME] \[CONTAINER NAME]=nginx:1.7.1`
    - 위 두 방식으로 application의 이미지를 업데이트할 수 있지만 둘의 definition file이 다름
    - 과정: 자동으로 ReplicaSet 생성>Replicas에 맞는 pod 생성>deployment가 새 replicaset을 만들고 container 배포

5. Rollback

- application을 업그레이드하면 잘못된 부분 포착 가능성. 업그레이드할때 사용하는 새버전 빌드에 문제 존재 => 업데이트 Rollback
- Kubernetes deployment는 이전 revision으로 rollback해줌. 변경된 것을 취소하려면 `k rollout undo deployment/[DEPLOYMENT NAME]`

6. Summarize Commands

- `Create`
    - `k create -f [FILE NAME]`
- `Get`:
    - `k get deployments`
- `Update`
    - `k apply -f [DEPLOYMENT NAME]`
    - `k set image deployment/[DEPLOYMENT NAME] [기존 IMAGE]=[변경하는 IMAGE]`
- `Status`
    - `k rollout status deployment/[DEPLOYMENT NAME]`
    - `k rollout histroy deployment/[DEPLOYMENT NAME]`
- `Rollback`
    -  `k rollout undo deployment/[DEPLOYMENT NAME]`

## Updating a Deployment

1. Deployment 생성 및 rollout status와 history 확인

`kubectl create deployment nginx --image=nginx:1.16`

`kubectl rollout status deployment nginx`

`kubectl rollout history deployment nginx`

2. `--revision` flag 사용

- 원하는 revision 조회

`kubectl rollout history deployment nginx --revision=1`

출력
```
deployment.extensions/nginx with revision #1
 
Pod Template:
 Labels:    app=nginx    pod-template-hash=6454457cdb
 Containers:  nginx:  Image:   nginx:1.16
  Port:    <none>
  Host Port: <none>
  Environment:    <none>
  Mounts:   <none>
 Volumes:   <none>
```

3. `--record` flag 사용

`kubectl set image deployment nginx nginx=nginx:1.17 --record` 로 "change-cause" field에 해당 명령어 기록

`kubectl rollout history deployment nginx`하면 출력이

```
deployment.extensions/nginx
 
REVISION CHANGE-CAUSE
1     <none>
2     kubectl set image deployment nginx nginx=nginx:1.17 --record=true
master $
```

4. `--record` flag 2개 사용

`kubectl edit deployments. nginx --record`

`kubectl rollout history deployment nginx`하면 출력이

```
REVISION CHANGE-CAUSE
1     <none>
2     kubectl set image deployment nginx nginx=nginx:1.17 --record=true
3     kubectl edit deployments. nginx --record=true
```

5. `undo`

`kubectl rollout history deployment nginx`

`kubectl rollout history deployment nginx --revision=3`

`kubectl rollout history deployment nginx --revision=1`

`kubectl rollout undo deployment nginx --to-revision=1`

- `--to-revision` 사용으로 배포를 생성한 첫 번째 이미지와 함께 롤백됨

## Demo - Deployments

1. `--record` 없이

- 1. `k create -f deployment-definition.yaml` : deployment 생성

- 2. `k rollout rollout status deployment/myapp-deployment`
- 3. `k rollout rollout history deployment/myapp-deployment`

    - 새로운 Kubernetes deployment가 생성되거나 기존 deployment가 업그레이드될 때마다 새로운 rollout 생성
        - `rollout`은 백엔드에서 컨테이너를 배포하는 프로세스 => `k rollout status`에서 한 번에 하나의 pod가 생성되는 것 확인 가능
    - 새 출시 버전이 생성될 때마다 새 deployment revision 생성 => `k rollout history`에서 확인 가능
        - history에서 CHANGE-CAUSE none

- 4. `k delete deployment myapp-deployment` : 모두 삭제

2. `--record` 있이(앞의 차이를 제외하고는 같은 코드 실행)

- 1. `k create -f deployment-definition.yaml --record` : deployment 생성
- 2. `k rollout rollout status deployment/myapp-deployment` : 출시상태 알아보기
    - 한 번에 한 pod 씩 배포
- 3. `k rollout rollout history deployment/myapp-deployment`
    - history에서 CHANGE-CAUSE에 명령어 존재

3. Image upgrade 또는 downgrade (definition file 수정 방식)

- 기존 pod-definition에서 image는 nginx라 nginx:latest였는데, nginx:1.12로 업그레이드
- 1. pod-definition 수정
- 2. `k apply -f deployment-definitino.yml` 적용(배포 업데이트)
- 3. `k rollout rollout status deployment/myapp-deployment`
- 4. `k get deployments` 또는 `k describe deployment`로 이미지 업데이트 확인. `describe` 명령어에서 Events 확인 가능
- 5. `k rollout rollout history deployment/myapp-deployment`에서 revision 1,2 확인

4. Image upgrade 또는 downgrade (명령어 사용 방식)

- 위에서 변경한 nginx:1.12를 nginx:1.12-perl로 업그레이드
- 1. `k set image deployment/myapp-deployment nginx-container=nginx:1.12-perl`
    - `k set image deployment/[DEPLOYMENT NAME] [CONTAINER NAME]=[변경하려는 IMAGE VERSION]`
- 2. `k rollout rollout status deployment/myapp-deployment`
- 3. `k rollout rollout history deployment/myapp-deployment`
    - 위에서 본 history에 하나 추가. REVISION 총 3개
        - revision1은 nginx(latest), revision2는 nginx:1.12, revision3는 nginx:1.12-perl
- 4. `k describe deployments`로 이미지 nginx:1.12-perl 확인 

5. 마지막 버전에 문제가 있다고 가정 => Rollback

- 1. `k rollout undo deployment/myapp-deployment`
- 2. `k rollout rollout status deployment/myapp-deployment`
- 3. `k rollout rollout history deployment/myapp-deployment`
    - revision이 1,3,4만 존재. 현재는 revision 4이며 revision2에서 사용한 definition file과 image 모두 사용
- 4. `k describe deployments`하면 image가 nginx:1.12로 이전의 revision2임을 알 수 있음

6. 오류가 있다고 가정: 존재하지 않는 nginx Image 제공 

- 1. definition file에서 image를 nginx:1.5-err로 설정>`kubectl apply -f deployment-definition.yml --record`
- 2. `k rollout status deployment/myapp-deployment`
    - 3개의 replicas가 준비되었다고 나오고 종료
    - `k get deployments`에서 DESIRED는 6이지만 UP-TO-DATE는 3인 상황 발생 
    - `k get pods`에서 실제로 작동하는 예전 버전과 현재 사용 불가능한 새 버전 존재 => DockerHub에서 이미지를 가져올수없음(ImagePullBackOff). 이전 버전의 instance를 하나 지우고 새 버전의 instance를 생성할 때 문제 발생해 이전 버전의 instance 삭제 중단
- 3. `k rollout rollout history deployment/myapp-deployment`
    - revision5 생성
- 4. `k rollout undo deployment/myapp-deployment`
    - 이미지가 잘못되었으므로 rollback 진행
- 5. `k rollout rollout history deployment/myapp-deployment`
    - revision4가 사라지고 그 버전으로 revision6 생성
- 6. 이후 pod와 deployment를 보면 pod는 총 6개, deployment의 image는 nginx:1.12임을 확인할 수 있음

## Deployment Strategy - Blue Green

1. Deployment Strategy
    - Recreate: 기존의 instance 한 번에 모두 삭제 후 새로운 버전의 instance 생성
    - RollingUpdate: instance 하나씩 삭제 후 생성 반복

2. Blue-Green

- 새 버전(Green)이 구 버전(Blue)과 함께 배포
- traffic은 구 버전으로 모두 전송. 이 시점에 Green에서는 테스트가 이루어짐. 모든 테스트가 통과하면 traffic을 새 버전으로 한 번에 전환
- Istio 같은 서비스 정책으로 잘 구현됨

3. Blue-Green 구현

- 1. 서비스로 배포된 Deployment-blue(구버전) 존재

`myapp-blue.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-blue
    labels:
        app: myapp
        type: front-end
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                version: v1
            spec:
                containers:
                    - name: app-container
                      image: myapp-image:1.0
    replicas: 5
    selector:
        matchLabels:
            version: v1
```
- 2. traffic 경로를 위한 Service 생성
`service-definition.yaml`
```
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        version: v1
```
- 3. Service와 Pod를 연관짓기 위해 pod에 label 붙이고 selector에서 같은 label 사용 => `version:v1`
- 4. Green deployment 배포 => `version:v2`
    - blue 파일이랑 metadata>name, spec>tempalte>labels>version, spec>tempalte>spec>containers>image ,spec>selector>matchLabels>version 4개 다름
`myapp-green.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-green
    labels:
        app: myapp
        type: front-end
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                version: v2
            spec:
                containers:
                    - name: app-container
                      image: myapp-image:2.0
    replicas: 5
    selector:
        matchLabels:
            version: v2
```
- 5. 모든 테스트가 통과되면 Service에서 traffic을 green deployment로 routing => selector를 v1에서 v2로 변경. service가 green deployment의 pod로 전환
`service-definition.yaml`
```
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        version: v2
```


## Deployment Strategy - Canary

1. Canary

- 새 버전을 배포해 소량의 traffic만 경로 지정
- 대부분의 traffic이 구 버전으로 전송되지만 새 버전으로 전송되는 소량의 traffic 존재. 이 시점에서 테스트를 하고 모든 것이 괜찮으면 기존 deployment를 새로운 버전의 application으로 업그레이드>canary deployment 제거
- 회전 업그레이드 전략

2. Canary 구현

- 1. 기존 deployment인 primary deployment 존재. pod5개
`myapp-primary.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-primary
    labels:
        app: myapp
        type: front-end
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                version: v1
                app: front-end
            spec:
                containers:
                    - name: app-container
                      image: myapp-image:1.0
    replicas: 5
    selector:
        matchLabels:
            app: front-end
```
- 2. traffic 경로를 위한 service 생성

`service-definition.yaml`
```
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        app: front-end
```
- 3. service와 pod의 연관을 위해 label과 selector 지정 => `version: v1`
- 4. 두번째 deployment인 canary deployment 배포
    - replicas 수가 다름, image, name
`myapp-primary.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-canary
    labels:
        app: myapp
        type: front-end
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                version: v2
                app: front-end
            spec:
                containers:
                    - name: app-container
                      image: myapp-image:2.0
    replicas: 1
    selector:
        matchLabels:
            app: front-end
```
    - Canary 배포시 핵심
    - (1) traffic이 동시에 두 버전으로 가길 원함 => 두개의 deployment에 일반적인 label 붙임 `app: front-end`
    - (2) 버전2로 약간의 traffic만 이동시키면 됨 => 서비스가 모든 pod에 traffic을 동등하게 분배하기 때문에 v1이 83%, v2가 17% 차지하도록 함
- 5. 테스트 수행하고 새 버전에 확신이 생기면 primary deployment에서 pod version 업그레이드하고 canary 모두 삭제

3. Canary 주의사항

- 각 deployment 사이의 traffic 분할에 대한 제한 존재
    - traffic 분할은 항상 각 deployment에 참여하는 pod 수에 따라 결정
    - Istio는 traffic의 정확한 비율 정의 가능. pod 수에 좌우되지않음

## Jobs

1. RestartPolicy

- Pod의 기본 동작은 container가 계속 작동하도록 재시작하는 것 => `기본 RestartPolicy:Always`
    - pod가 나갈때마다 container 재시작

- RestartPolicy
    - (기본)Always
    - Never
    - OnFailure

2. Jobs

- 할당된 작업을 하고 일이 성공적으로 끝나도록 보장할 수 있는 관리자 필요
- ReplicaSet으로 여러 개의 pod 생성을 보장하는 반면, Jobs는 해당 pod 세트를 실행하는 작업은 주어진 작업을 완료하는 데 사용됨

3. Job definition file

`job-definition.yaml`
```
apiVersion: batch/v1
kind: Job
metadata:
    name: math-add-job
spec:
    completions: 3
    parallelism: 3
    template:
        spec:   #아래 pod-definition의 spec과 동일
            containers:
                - name: math-add
                image: ubuntu
                command: ['expr','3','+','2']
            restartPolicy: Never
```

`pod-definition.yaml`
```
apiVersion: v1
kind: Pod
metadata:
    name: math-pod
spec:
    containers:
        - name: math-add
          image: ubuntu
          command: ['expr','3','+','2']
    restartPolicy: Never
```

- `k create -f job-definition.yaml`으로 job 생성
- `k get jobs`
- `k get pods`에서 RESTARTS가 0
- `k logs [POD NAME]`로 job의 command 출력 확인
- `k delete job [JOB NAME]`
- `completions`를 설정해 몇 번 성공하기를 바라는지 지정. 하나의 pod가 완료되면 다음 pod가 순차적으로 생성. 중간에 오류가 나도 completion을 채울 때까지 pod 생성

4. Parallelism

- pod를 순차적으로 만드는 대신 병렬적으로 만들 수 있음
- 아래 코드는 병렬적으로 한 번에 3개의 pod를 생성하고 이후에는 completions가 3이 될 때까지 pod를 1개씩 생성

`job-definition.yaml`
```
apiVersion: batch/v1
kind: Job
metadata:
    name: math-add-job
spec:
    completions: 3
    parallelism: 3
    template:
        spec:   #아래 pod-definition의 spec과 동일
            containers:
                - name: math-add
                image: ubuntu
                command: ['expr','3','+','2']
            restartPolicy: Never
```

## CronJobs


1. CronJobs

- Jon이 실행하는 작업을 `주기적`으로 실행
- schedule 옵션은 cron 형식 문자열
- jobTemplate은 실행되어야 할 실제 작업
- spec은 총 3개: cronjob, job, pod

2. CronJobs definition file
`cron-job-definition.yaml`
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
    name: reporting-cron-job
spec:
    schedule: "*/1 * * * *"
    jobTemplate:
        spec:
            completions: 3
            parallelism: 3
            template:
                spec:
                    containers:
                        - name: reporting-tool
                          image: reporting-tool
                    restartPolicy: Never
```

## Labs 실습

1. Labels and Selectors

Q1.

- Sol: 숫자를 세는 경우 마지막에 `| wc -l`을 붙이면 되는데, 일반적으로 `k get pods`와 같은 경우 맨 위에 header 존재. 따라서 header를 뺀 상태로 뒤에 붙여써야함 `k get pods --selector env=dev --no-headers | wc -l`

Q4. 

- `k get pods --selector key1=value1&&key2=value2`와 같이 &&로 묶으면 하나라도 해당하는 경우 모두 출력. 모두 해당하는 pod를 찾는 경우 `&&`가 아닌 `,`로 연결

2. Rolling Updates & Rollbacks

Q4.

- Sol: `./curl-tet.sh`를 하면서 배포 테스트

Q8.

- `k edit deployments frontend`한 후 코드 수정하면 자동 배포 또는 `k set image deployment frontend simple-webapp=kodekloud/webapp-color:v2`

Q11.

- spec>strategy> `type: Recreate`

3. Deployment strategies

Q5. 

- Sol: traffic을 20% 미만으로 배포를 구성하라했는데, traffic은 deployment의 pod 수와 비례하므로 pod 수를 1로 줄임 `k scale deployment --replicas 1 frontend-v2`
    - `k scale deployment --replicas 1 [DEPLOYMENT NAME]`

4. Jobs and CronJobs

Q1.

- Sol: `k logs throw-dice-pod`로 주사위 값 조회 가능

Q2.

- Sol: `k create job throw-dice-job --image kodekloud/throw-dice --dry-run=client -o yaml > throw-dice-job.yaml`
- Sol: `backoffLimit`은 pod의 실패 제한값. backoffLimit이 3이면 3번 실패하면 멈추는 Job
