## Services

1. Service

- application 안밖의 다양한 `구성요소 간의 통신`을 가능하도록 함
- application을 다른 application 또는 사용자와 연결하는데 도움을 줌
- pod group 간의 연결을 가능하게 함
    - application엔 다양한 섹션을 실행하는 pod group 존재 
    - 프론트엔드 처리하는 그룹,백엔드 프로세스를 실행하는 또다른 그룹 등
    - 외부 데이터 소스에 연결된 third group 존재
- Service는 프론트엔드 applicatoin을 사용자가 사용할 수 있도록 하며, 프론트엔드와 백엔드 pod 간의 통신을 돕고 외부 데이터 소스와의 연결을 구축하는데 도움을 줌


2. Service 등장

- 지금까지 내부 네트워크를 통해 pod가 서로 소통하는 법을 이야기 함
- Service를 사용하면 외부 통신 가능
- 기존 Setup
    - Kubernetes node의 IP 주소는 `192.168.1.2`. 노트북도 같은 network에 존재하며 IP 주소는 `192.168.1.10`
    - 내부 pod의 network는 `10.244.0.0` 범위 내에 존재하고 pod는 IP 주소가 `10.244.0.2`
    - 이 경우 User는 `분리된 network`에 존재하기에 10.244.0.2의 pod에 접근할 수 없음
    - 접근 방법
        - `curl http://10.244.0.2`로 SSH 접근
        - or 노드에 GUI가 있다면 브라우저를 실행해 웹페이지 접근 가능
- Node에 SSH를 사용하지 않고 node의 IP를 액세스함으로써 노트북에서 웹 서버에 접근하길 원함 => `Service`

3. Service Types
- NodePort
    - Service가 node의 port에 내부 pod port를 access
- ClusterIP
    - Service는 내부에 가상 IP를 만들어 frontend server set와 backend server set 같은 다양한 service 간의 통신을 가능하게 함
- LoadBalancer
    - 지원되는 cloud provider에서 우리 application을 provision
    - ex. frontend 계층의 다양한 web server에 load 배포

4. Service Type -NodePort
- 3개의 port 존재
    - `targetPort`: 실제 web server가 실행 중인 pod의 port. 서비스가 요청을 전달하는 객체의 port. 할당하지 않는 경우 port와 동일하게 인식
    - `port`: service의 port. service는 node 안의 가상 server와 같음. cluster 내부에 고유한 IP 주소(=service cluster IP) 존재
    - `nodePort`: Node에 존재하는 port. 외부 웹 서버에 액세스할 때 사용하며 유효 범위(30,000~32,767)에만 존재. 할당하지 않은 경우 유효 범위 내에서 자동으로 할당
- service definition file
    - Service의 spec에는 다른 object와 달리 `type`과 `ports` 섹션 존재. 다른 object와 비슷하게 `selector` 존재
        - type은 service의 type
        - ports는 배열이므로 - 필요. 하위 섹션으로 targetPort, port, (nodePort인 경우)nodePort 존재
        - selector로 service와 원하는 pod 매핑
`service-definition.yml`
```
apiVersion: v1
kind: Service
metadata:
    name: myapp-service
spec:
    type: NodePort
    ports:
        - targetPort: 80    
          port: 80
          nodePort: 30008
    selector:
        app: myapp
        type: front-end
```
- kubectl 명령어
    - `kubectl create -f service-definition.yml`
    - `kubectl get services`
        - TYPE에는 service type, PORT(S)에는 port 나타남
        - nodePort service의 경우 PORT(S)가 [port]/[nodePort]
    - `curl http://192.168.1.2:30008`
        - `curl [nodeIP]:[nodePort]` 하면 nodePort를 이용해 웹 서버 액세스 가능

- **Pod가 여러 개인 경우** service의 selector가 label을 가리키기에 여러 pod 접근 가능

> 여러 pod에 부하 분산을 위해 어떤 알고리즘을 쓰는지 궁금하다면 `Random SessionAffinity` 사용

- **다중 Node에 pod가 분산된 경우** cluster 내의 개별 node pod에 application 존재. service를 생성할 때 kubernetes는 자동으로 모든 node를 가로질러 cluster 내의 모든 node에 같은 node 매핑. cluster에 있는 nodeIp와 같은 port 번호를 사용해 해결 가능. 이 경우엔 30008

- 요약하면 service는 어떤 상황이든 똑같이 생성. pod가 제거되거나 추가되어도 service는 자동으로 업데이트되어 유연하고 적응적 

## Services - ClusterIP

## Ingress Networking

## Article: Ingress

## FAQ - What is the rewrite-target option?

## Network Policies

## Developing network policies

## Labs 실습