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

## Labs 실습

1. Multi-Container Pods

2. Init Containers