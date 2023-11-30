## Multi-Container Pods

1. Microservice

- 대형 Monolithic application을 micro service라는 하위 구성 요소로 분리하는 아이디어는 독립적이고 작은 재사용 가능한 코드 세트를 개발 및 배포할 수 있도록 해줌

- 본 아키텍쳐는 `scale up, down`을 돕고, 전체 application을 수정하는 것 대신 개별 서비스를 수정할 수 있음

- 때로는 Web Server와 Log Agent 서비스와 같이 두 가지 서비스가 협업에 필요할 수 있음
    - 하나의 Web Server 인스턴스 당 Log Agent 인스턴스 하나 필요
    - 두 서비스 코드를 합쳐서 망치면 안 되고, 각각 다른 기능을 대상으로 하는데도 여전히 따로 개발되고 배포되길 원함
    - 두 가지 기능만 함께 작동하면 됨
    - scale up, down을 함께 할 수 있는 web server 인스턴스당 하나의 agent 필요 => 같은 수명주기를 공유하는 Multi-Container Pod: 함께 생성되고 함께 파괴됨

2. Multi-Container Pods

- 같은 수명주기를 공유
    - 함께 생성되고 함께 파괴됨
- 같은 네트워크 공간 공유
    - 서로를 localhost라고 부를 수 있고 같은 저장소 volume 접근 가능
- Multi-Container Pods를 통해 통신이 가능하도록 Pod 사이의 Volume 공유나 서비스를 설정하지 않아도 됨

- Multi-Container Pods 생성
    - 1. pod-definition 파일(pod-definition.yaml)에 새 container 정보 추가
        - pod 정의 파일에서 spec>containers는 array(배열)인 이유는 Pod에 여러 개의 Container를 두기 위해서
    `pod-definition.yaml`
    ```
    apiVersion: v1
    kind: Pod
    metadata:
        name: simple-webapp
        labels:
            name: simple-webapp
    spec:
        containers:
            - name: simple-webapp
              image: simple-webapp
              ports:
                - containerPort: 8080
            - name: log-agent
              image: log-agent
    ```

3. Design Patterns

: Multi-Container Pods를 설계할 때 공통되는 패턴

- 1. Sidecar Container
    - Web Server와 함께 Logging Agent를 배포해 log를 모아 중앙 로그 서버로 전송
    - ex. 로그인 서비스
    - Multiple application이 다양한 포맷으로 log를 생성한다면 중앙 로그 서버에서 다양한 포맷을 처리하기 힘듬 => 로그를 중앙 서버로 보내기 전에 `로그를 일반 포맷으로 변환`하고 싶음 => `Adapter Container 배포`
- 2. Adapter Container
    - 중앙 서버로 전송하기 전에 로그 처리

- 3. Ambassador Container
    - application은 다양한 단계의 데이터베이스와 통신: ex. Dev, Test, Prod
    - application 배포 환경에 따라 코드의 데이터베이스 연결을 수정해야함
    - application이 항상 로컬 호스트에서 데이터베이스를 참조하도록 별도의 container에 아웃소싱 가능 
    - 새 container는 해당 DB에 `proxy` 요청을 함 => Ambassador Container

- Multi-container pod를 설계할 때 패턴이 다름
- pod-definition 파일을 이용해 그들을 구현할 때 항상 동일. pod-definitino 파일 안에 container가 여러 개 존재할 뿐

## Init Containers

1. Init Containers

- multi-container pod에서 각 container는 Pod의 라이프사이클만큼 길게 동작하는 것이 예상됨. 예를 들어 multi-container pod에서 web application과 log agent가 존재하는 경우, 두 contianer는 항상 활성화 상태를 유지할 것으로 예상되며 하나라도 오류가 발생하면 pod가 재실행됨

- But 때로는 container 안에서 완료까지 진행되는 프로세스를 원할 수 있음. 예를 들어 메인 web application이 repository에서 코드나 binary를 끌어오는 프로세스의 경우 pod가 처음 만들어질 때 한 번만 실행되는 작업. 또다른 예로 application이 시작되기 전 외부 서비스나 데이터베이스가 올라오기를 기다리는 프로세스 존재 => `initContainers` 도입

- initContainers는 iniContainers 섹션 내에 지정된 것을 제외하고는 다른 container와 마찬가지로 pod에 구성

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ;']
```

- Pod가 처음 생성될 때 initContainer가 실행되고 initContainer의 프로세스는 application을 호스팅하는 실제 container가 시작되기 전에 완료 상태로 실행되어야함

- multi-pod container의 경우와 같이 여러 개의 initContainer를 구성할 수 있음. 이 경우 각 initContainers는 한 번에 하나씩 순차적으로 실행됨

- initContainers 중 하나라도 완료하지 못하면 Kubernetes는 initContainers가 성공할 때까지 Pod를 반복적으로 재시작

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']

```

## initContainer 참고
<https://kubernetes.io/docs/concepts/workloads/pods/init-containers/>

## Labs 실습

1. Multi-Container Pods

Q3. Create a multi-container pod with 2 containers. Use the spec given below. If the pod goes into the crashloopbackoff then add sleep 1000 in the lemon container.

- Sol: `k run yellow --image=busybox --dry-run=client -o yaml`으로 코드 확인 후 `k run yellow --image=busybox --dry-run=client -o yaml > yellow.yml`으로 파일 생성 > 파일 수정 > 선택사항으로 sleep 1000은 `command: ["sleep","1000"]` 또는 `command: - sleep - 1000` 

Q4. 

- Sol: `elastic-stack` namespace에는 app, elastic-search, kibana pod 존재. Elastic-search는 데이터 수집 장소를 탄력적으로 검색 가능. 메트릭 로그일 수 있고 application에서 수집한 로그일 수도 있음. Kibana는 사용자나 관리자가 로그를 필터링할 때 사용하는 대시보드. 목표는 application의 일지를 elastic-search로 옮기고 elastic-search에서 검색하면 Kibana 대시보드에서 일지를 볼 수 있도록 함

Q5. Once the pod is in a ready state, inspect the Kibana UI using the link above your terminal. There shouldn't be any logs for now.


We will configure a sidecar container for the application to send logs to Elastic Search.

NOTE: It can take a couple of minutes for the Kibana UI to be ready after the Kibana pod is ready.

You can inspect the Kibana logs by running:

`kubectl -n elastic-stack logs kibana`

Q7. The application outputs logs to the file /log/app.log. View the logs and try to identify the user having issues with Login. Inspect the log file inside the pod.

`kubectl -n elastic-stack exec -it app -- cat /log/app.log`

- Sol: application의 log 파일인 /log/app.log는 pod 내에 존재. Pod 내에 존재하는 log file 확인. `k logs app -n elastic-stack`으로 elastic-stack 네임스페이스에 존재하는 app이라는 pod의 log 확인 가능. `kubectl -n elastic-stack exec -it app -- cat /log/app.log`를 통해서도 동일한 로그 확인 가능

Q8.

- Sol: 위에서 log를 /log/app.log에 저장하는 것 확인 > pod를 연결하거나 설정해서 중심부(ElasticSearch)로 log 전송. 따라서 pod에 sidecar container 추가. sidecar의 이미지인 beat-configured는 자동으로 데이터가 전송되도록 하는 이미지 => Pod에는 이제 2개의 container가 들어가며 app container는 실제 application이 동작하는 container이고, sidecar container는 로그를 특정 경로나 해당 볼륨에 저장하는 container. volumeMounts 이름이 같으므로 같은 volume 사용하지만 container에서 해당 volume으로 가는 경로가 다름

Q9. 

- Sol: Kibana UI로 이동해 (1) index pattern 생성.index pattern으로 데이터 검색 => *로 설정+configure settings에서 I don't want to use the Time Filter (2) Discover에서 log 확인

2. Init Containers

Q3. 

- pod를 describe해 보는 경우 container마다 state 존재

Q6. 

- pod의 status는 pod describe 시 맨 위에 나타남

Q7. How long after the creation of the POD will the application come up and be available to users?

- Sol: pod 생성 후 application이 시작할 때부터 사용자가 사용하는 사이의 시간은 initContainer에 의해 결정됨. sleep 600 + sleep 1200 하면 토탈 1800초, 즉 30분 소요

Q8.

- Sol: initContainer name을 red-initcontainer로 설정해야함