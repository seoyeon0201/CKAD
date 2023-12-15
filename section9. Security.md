## Authentication, Authorization and Admission Control

#### Secure Hosts

- Host는 cluster 생성

- Host에 대한 모든 액세스는 보안되어야 함
    - Password based authentication disabled
    - SSH Key based authentication

#### Secure Kubernetes

1. kube-apiserver

- `apiserver 자체에 대한 액세스 제어`
- kube-apiserver
    - kubectl utility 또는 API에 직접 접근하므로써 상호작용해 클러스터에서 작업 수행

- 1. Who can access?
    - `Authentication`
    - apiserver에 인증할 수 있는 다양한 방법 존재
        - Files - Username and Passwords
        - Files - Username and Tokens
        - Certificates
        - External Authentication providers - LDAP
        - Service Accounts

- 2. Whan can they do?
    - `Authorization`
    - 해당 사용자가 권한이 있는 작업을 하는가
        - RBAC Authorization
        - ABAC Authorization
        - Node Authorization
        - Webhook Mode

- TLS Certificates
    - 클러스터와의 모든 `통신 구성요소`는 node에서 실행되고 TLS 암호화로 보안 
        - etcd cluster, kube controller manager, kube scheduler, kube proxy, kubelet
    - 그들 사이의 인증서 설정 방법 존재

- Network Policy
    - `cluster 내의 application` 간의 통신

## Authentication

#### Kubernetes에 액세스하는 User
- kubernetes cluster는 다중 노드에서 물리적, 가상적으로 다양한 구성요소를 가지고 구성요소와 함께 동작하는 구성요소로 구성
- `Admin`은 관리 업무를 수행하기 위해 클러스터에 액세스
- `Developer`는 application을 테스트 또는 배포하기 위해 클러스터에 액세스
- `End User`는 배포된 application을 액세스
- `Bots`는 end user와 통합 용도로 클러스터에 액세스하는 타사 application

- 본 섹션에서는 `Authentication`을 통해 Kubernetes `Cluster에 대한 액세스 권한`을 확보에 초점

#### End User

1. Application End User
- application 자체에서 내부적으로 end user에 대해 security 관리
- 해당 user는 제외

#### Accounts - User Accounts 
-`Admin & Developer`

1. Kubernetes는 user account를 직접 관리하지 않음
- 외부 소스에 의존
    - user detail이 있는 file, ID service (ex. LDAP)

`kubectl create user user1` 또는 `kubectl list users` 불가능 

2. kube-apiserver

- user의 모든 요청은 kube-apiserver로 전송
- kube-apiserver
    - 1. Authenticate User: `Auth Mechanism`을 이용해 사용자 인증
        - `Static Password File`에는 사용자 이름과 비밀번호 
        - `Static Token File`에는 사용자 이름과 토큰
        - `Certificates` 인증서
        - `Identity Services` 타사 인증 프로토콜에 연결. ex. LDAP, Kerberos 

    - 2. Process Request: 요청 처리

- `Auth Mechanism` - Static Password File, Static Token File
    - 사용자이름, 암호, 토큰을 텍스트에 저장하는 해당 방식은 불안정하므로 권장하지 않음
    - 1. Static Password File
        - csv 파일에 비밀번호, 사용자 이름, id 존재. id 뒤에 group을 할당할 수 있음
        `/tmp/users/user-details.csv`
        ```
        password123, user1, u0001, group1
        password123, user2, u0002, group1
        password123, user3, u0003, group2
        password123, user4, u0004, group2
        password123, user5, u0005, group2
        ```
        - `kube-apiserver.service`에 `--basic-auth-file=user-details.csv` 추가 > apiserver 재시작
        - `/etc/kubernetes/manifests/kube-apiserver.yaml`에 아래와 같이 추가 > 파일 업데이트해 재시작
        ```
        apiVersion: v1
        kind: Pod
        metadata:
            creationTimestamp: null
            name: kube-apiserver
            namespace: kube-system  #kube-apiserver의 namespace는 kube-system 설정
        spec:
            containers:
            - command:
                - kube-apiserver
                <content-hidden>
                - --basic-auth-file=/tmp/users/user-details.csv #코드 추가
                image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
                name: kube-apiserver
                volumeMounts:
                - mountPath: /tmp/users #csv 파일의 위치가 mountPath로 지정
                name: usr-details
                readOnly: true  #readOnly로 지정
            volumes:
            - hostPath:
                path: /tmp/users    #csv 파일의 위치가 path로 지정
                type: DirectoryOrCreate
                name: usr-details
        ```
        - 사용자의 yaml 파일에 아래 Role, RoleBinding 추가
        ```
        ---
        kind: Role
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
        namespace: default
        name: pod-reader
        rules:
        - apiGroups: [""] # "" indicates the core API group
        resources: ["pods"]
        verbs: ["get", "watch", "list"]
        
        ---
        # This role binding allows "jane" to read pods in the "default" namespace.
        kind: RoleBinding
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
        name: read-pods
        namespace: default
        subjects:
        - kind: User
        name: user1 # Name is case sensitive
        apiGroup: rbac.authorization.k8s.io
        roleRef:
        kind: Role #this must be Role or ClusterRole
        name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
        apiGroup: rbac.authorization.k8s.io
        ```

        - 사용: `curl -v -k https://localhost:6443/api/v1/pods -u "user1:password123""`
        

    - 2. Static Token File
        - csv 파일에 token, 사용자 이름, id, group name 존재
        - `kube-apiserver.service`에 `--token-auth-file=token-details.csv` 추가 > apiserver 재시작
        - 위의 static password file과 유사하게 설정
        - 사용: `curl -v -k https://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer [TOKEN]"`로 authorization bearer token 지정

#### Accounts - Service Accounts 
`Bots`

- Kubernetes가 직접 관리
`kubectl create serviceaccount sa1` 또는 `kubectl get serviceaccount` 가능


## KubeConfig

#### 사용자를 위한 인증서

- cluster가 my-kube-playground인 경우
- 클라이언트가 인증서 파일을 어떻게 사용하는지
- 쿠버네티스 REST API를 쿼리하기 위한 키를 어떻게 사용하는지

- `curl` 사용
```
curl https://my-kube-playground:6443/api/v1/pods --key admin.key --cert admin.crt --cacert ca.crt
```

- `kubectl` 사용
```
kubectl get pods --server my-kube-playground:6443 --client-key admin.key --client-certificate admin.crt --certificate-authority ca.crt
```

#### config 파일

- curl 또는 kubectl로 매번 옵션 추가하기 힘듬 => `config` 파일 생성

1. Config 생성 & 사용

`$HOME/.kube/config`
```
--server my-kube-playground:6443
--client-key admin.key
--client-certificate admin.crt
--certificate-authority ca.crt
```

- `kubectl` 사용
`kubectl get pods --kubeconfig config`

2. KubeConfig File 

- Config file은 3가지 섹션 존재 : `Clusters`, `Contexts`, `Users`

- Cluster
    - 액세스가 필요한 다양한 Kubernetes cluster
    - `Development`, `Production`, `Google`

- User
    - cluster에 액세스 권한이 있는 사용자 계정
    - user는 다른 cluster에서 다른 권한을 가질 수 있음
    - `Admin`, `Dev User`, `Prod User`

- Context
    - Cluster와 User가 만나 어떤 user가 어떤 cluster에 액세스하기 위해 사용될지 정의
    - 새로운 user를 생성하고 액세스 권한을 설정하는 것이 아니라, `기존의 user`를 어떤 cluster에 사용할지 정의 
        - 사용자 인증서와 서버 주소를 지정할 필요 X
    - ex1. `Admin@Production`은 Admin user를 이용해 Production cluster에 액세스
    - ex2. `Dev User@Google`은 application 테스트 배포를 위해 dev user가 Google cluster에 액세스

- 위에서 본 예시 config file에서 첫번째 --server와 --certificate-authority는 Cluster, 나머지는 User, 해당 파일에서 Context 이루어짐

`$HOME/.kube/config`
```
--server my-kube-playground:6443
--client-key admin.key
--client-certificate admin.crt
--certificate-authority ca.crt
```

3. KubeConfig File 예시

```
apiVersion: v1
kind: Config
clusters:
    - name: my-kube-playground
      cluster:
        certificate-authority: ca.crt
        server: https://my-kube-playground:6443
contexts:
    - name: my-kube-admin@my-kube-playground
      context:
        cluster: my-kube-playground
        user: my-kube-admin
users:
    - name: my-kube-admin
      user:
        client-certificate: admin.crt
        client-key: admin.key
```
- 현재 context 지정 가능
    - values는 hidden 상태

`my-custom-config`
```
apiVersion: v1
kind: Config
current-context: dev-user@google    #현재 context 지정
clusters:
    - name: my-kube-playground
    - name: development
    - name: production
    - name: google
contexts:
    - name: my-kube-admin@my-kube-playground
    - name: dev-user@google
    - name: prod-user@production
users:
    - name: my-kube-admin
    - name: admin
    - name: dev-user
    - name: prod-user
```

4. kubectl config

- 현재 config 파일 조회 `kubectl config view`
- 특정 config 파일 조회 `kubectl config view --kubeconfig=my-custom-config`
- 현재 context 변경 `kubectl config use-context prod-user@production`

5. Cluster Namespace 지정

- contexts.context에 namespace 지정해 해당 cluster의 어떤 namespace에 액세스할지 지정

```
apiVersion: v1
kind: Config
clusters:
    - name: my-kube-playground
      cluster:
        certificate-authority: ca.crt
        server: https://my-kube-playground:6443
contexts:
    - name: my-kube-admin@my-kube-playground
      context:
        cluster: my-kube-playground
        user: my-kube-admin
        namespace: finance  #namespace 지정
users:
    - name: my-kube-admin
      user:
        client-certificate: admin.crt
        client-key: admin.key
```

#### Certificates in KubeConfig

- 위 config file에 나온 `ca.crt`, `admin.crt`, `admin.key`가 certificates
- 경로 지정해 인증서를 지정하는 것이 좋지만, 다르게 지정할 수 있음
    - 경로 지정: clusters.cluster.certificate-authority에 `/etc/kubernetes/pki/ca.crt`
    - crt file 내용 사용: crt 파일을 64로 인코딩해 clusters.cluster.certificate-authority-data에 인코딩한 값 넣기
        - 인코딩: `cat ca.crt | base64`
        - 디코딩(인증서 데이터가 든 파일을 보는 경우) : `echo "[인코딩값]" | base64 --d`

## API Groups

1. Kubernetes API

- 목적에 따라 여러 그룹으로 그룹화
    - `/version`: cluster version 확인
    - `/metrics`, `/healthz`: cluster의 상태 모니터링
    - `/logs`: application과 통합하는데 사용

2. /api 와 /apis

- `/api`는 core group, `/apis`는 named group

- `/api`
    - namespaces, pods, rc, events, endpoints, nodes, bindings, PV, PVC, configmaps, secrets, services


- `/apis`
    - /api보다 더 조직적
    - 모든 새로운 기능이 /apis에 해당
    - `API Groups`: /apps, /extensions, /networking.k8s.io, /storage.k8s.io, /authentication.k8s.io, /certificates.k8s.io
        - API Groups 안에는 `Resources` 존재
        - /apps -> /v1 -> /deployments, /replicasets, /statefulsets
        - /networking.k8s.io -> /v1 -> /networkpolicies
        - /certificates.k8s.io -> /v1 -> /certificatesigningrequests
            - Resources 안에는 `Verbs` 존재
            - /deployments -> list,get,create,delete,update,watch
    - verb를 통해 사용자에 대한 액세스 제어

- `curl http://localhost:6443 -k`로 사용 가능한 API Groups 리스트 조회 가능
    - 위 명령어로는 특정 API 버전을 제외하고는 액세스할 수 없음 => Authentication mechanism 지정 X
    - 1. 명령어에 authentication 넘겨줌
        - `curl http://localhost:6443 -k --key admin.key --cert admin.crt --cacert ca.crt`
    - 2. kubectl proxy client 시작
        - `kubectl proxy`로 proxy 실행. 이때 인증을 위한 자격 증명과 인증서 사용
        - `curl http://localhost:6443 -k`로 API Groups 리스트 조회
- `curl http://localhost:6443/apis -k |grep "name"`으로 모든 Reousrces 조회 가능


3. Kube proxy vs Kubectl proxy

- kube proxy
    - 클러스터 내 다양한 node에 걸쳐 서비스간의 연결을 가능하게 함

- kubectl proxy
    - HTTP proxy 서비스로 kubectl 유틸리티가 kube-apiserver에 액세스하기 위해 사용

## Authorization

- 이전까지는 컴퓨터 또는 사람이 어떻게 클러스터에 접속할 수 있는지 Authentication에 대해 다룸
- 본 섹션부터는 클러스터에 접근한 이후 어떤 작업을 할 수 있는지 권한 판단

#### Authorization 필요성

- Admin으로써 pod, node 등 모든 작업 가능
- 다른 admin, developer, tester monitoring service, cicd application도 접속 가능
- cluster에 접속할 계정(사용자 이름,암호,토큰)을 만들어 인증서 또는 serviceaccount 생성
- 모두가 같은 수준의 액세스를 가지길 원하지 않음

#### Authorization Mechanisms

1. Node Authorizer

- kube-apiserver는 사용자에 의해 액세스됨
- kube-apiserver는 관리 목적으로 클러스터 내 node의 kubelet에서도 액세스
    - kubelet은 kubeapiserver에 액세스해 service, endpoint, node, pod를 read하고 node status, pod status, event를 write
    - 이 요청을 `Node Authorizer`가 처리

- 이전에 certificate에 관해 얘기할 때 kubelet은 system node gorup의 일부이며 이름 앞에 system node를 붙여야 함
- system node name과 system node group으로 사용자가 요청하면 Node Authorizer가 승인해 authorization 부여
    - 클러스터 내의 액세스

2. ABAC

- 사용자나 사용자 그룹을 authorization 모음으로 연결
- policy를 생성해야하며 수동으로 수정하고 kube-apiserver를 다시 시작해야하므로 관리 어려움
- ex. developer는 pod를 조회, 생성, 삭제 가능
```
{"kind": "Policy", "spec": {"user": "dev-user", "namespace":"*", "resource":"pods", "apiGroup":"*"}}
{"kind": "Policy", "spec": {"user": "dev-user-2", "namespace":"*", "resource":"pods", "apiGroup":"*"}}
{"kind": "Policy", "spec": {"group": "dev-users", "namespace":"*", "resource":"pods", "apiGroup":"*"}}
{"kind": "Policy", "spec": {"user": "security-1", "namespace":"*", "resource":"csr", "apiGroup":"*"}}
```

3. RBAC

- 사용자를 직접 지정하지 않고 Role 생성해 지정

4. Webhook

- 외부에서 권한을 관리하고 싶을 때 시용
- ex. Open Policy Agent
    - 사용자에 대한 정보와 요구 사항 액세스 정보를 open policy agent로 넘기면, 사용자의 허용 여부 결정
5. 이외

- AlwaysAllow
- AlwaysDeny

#### mechanism 지정 방식

- kube-apiserver의 authorization-mode에서 설정 가능
- default는 AlwaysAllow
- authorization-mode를 여러개 설정하면 순서대로 지정할 수 있는데, 권한이 거절되면 다음 authorization mode로 넘어가며 승인이 완료된 사용자는 즉시 해당 리소스에 액세스 가능


## Role Based Access Controls

#### Role 생성

- `resourceNames`: 모든 resource에 대해서가 아닌 resource 중에서도 이름이 resourceName으로 지정된 resource만 접근 가능

`developer-role.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    name: developer
rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["list", "get", "create", "update", "delete"]
      resourceNames: ["blue","orange"]
    - apiGroupt: [""]
      resources: ["ConfigMap"]
      verbs: ["create"]
```

- `kubectl create -f develop-role.yaml`

#### 사용자와 Role 연결 - RoleBinding

- role과 rolebinding은 동일한 namespace 아래에 존재. 다른 namespace 원하는 경우 metadata.namespace 지정 가능
- `subjects` : 사용자 세부정보
- `roleRef` : role에 대해 상세히 제공
`devuser-developer-binding.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
    name: devuser-developer-binding
subjects:
    - kind: User
      name: dev-user
      apiGroup: rbac.authorization.k8s.io
roleRef:
    kind: Role
    name: developer
    apiGroup: rbac.authorization.k8s.io
```

- `kubectl create -f devuser-developer-binding.yaml`

#### kubectl RBAC

1. 기본 kubectl
- `kubectl get roles`
- `kubectl get rolebindings`
- `kubectl describe role developer`
- `kubectl describe rolebinding devuser-developer-binding`

2. 접근 권한 유무 확인
- `kubectl auth can-i create deployments`
- `kubectl auth can-i delete nodes`
- `kubectl auth can-i create deployments --as dev-user`
- `kubectl auth can-i create pods --as dev-user`
- `kubectl auth can-i create pods --as dev-user --namespace test`

## Cluster Roles

- 이전에 다룬 role과 rolebinding은 namespace 내에서 액세스 제어
- 리소스는 namespace 범위 또는 cluster 범위로 분류됨
    - ex. namespace dev, default, prod 중 하나의 리소스가 dev와 prod에 존재하는 경우 namespace 범위로 분류X

#### Namespace 범위 리소스

- pod, replicaset, job, deployment, service, secret, role, rolebinding, configmap, pvc
- `kubectl api-resources --namespaced=true`로 조회 가능
#### Cluster 범위 리소스

- node, pv, clusterrole, clusterrolebinding, certificatesigningrequests, namespaces

- `kubectl api-resources --namespaced=false`로 조회 가능

#### Clusterroles

1. 클러스터 범위 리소스에서만 차이가 있고 role과 동일

- namespace 전반에 걸친 리소스 액세스 제어

- Cluster Admin
    - Can view Nodes
    - Can create Nodes
    - Can delete Nodes

- Storage Admin
    - Can view PVs
    - Can create PVs
    - Can delete PVCs

2. ClusterRole 생성

`cluster-admin-role.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
    name: cluster-administrator
rules:
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["list","get","create","delete"]
```
- `kubectl create -f cluster-admin-role.yaml`

#### Clusterrolebinding

1. Clusterrolebinding 생성
`cluster-admin-role-binding.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
    name: cluster-admin-role-binding
subjects:
    - kind: User
      name: cluster-admin
      apiGroup: rbac.authorization.k8s.io
roleRef:
    kind: ClusterRole
    name: cluster-administrator
    apiGroup: rbac.authorization.k8s.io
```
- `kubectl create -f cluster-admin-role-binding.yaml`

## Admission Controllers

#### Securing Kubernetes

1. kubectl 사용 pod 생성 시

- `kubectl` > `kube apiserver` (`Authentication` > `Authorization` )  > `create pod`

- 요청이 kubectl을 통해 전송된 경우, kubeconfig 파일(.kube/config)은 인증서가 구성된 걸 알고 > `Authentication` 과정에서는 요청을 보낸 사용자를 식별하는데 책임이 있음 > `Authorization` 과정에서는 사용자가 이 작업을 수행할 수 있는 권한을 가지고 있는지 확인

2. Authorization에서 할 수 없는 작업

- 아래 작업인 경우에만 권한 부여
    - Only permit images from certain registry
    - Do not permit runAs root user
    - Only permit certain capabilities
    - Pod always has labels

3. Admission Controller 등장

#### Admission Controller

1. cluster 사용 방식을 강화하기 위한 더 나은 보안 조치 시행을 도움
    - 요청 자체를 변경하거나 포드가 생성되기 전 추가 작업 가능

- `AlwaysPullImages`, `DefaultStoragClass`, `EventRateLimit`, `NamespaceExists`, `NamespaceAutoProvision` 등

    - 존재하지 않는 namespace인 blue에서 pod를 생성하고 싶은 경우, `kubectl run nginx --image nginx --namespace blue` > namespace 존재하지 않는다는 오류 발생
        -  Authorization까지 거치지만 Admission Controller의 `NamespaceExists`에서 통과하지 못함
        
2. View Enabled Admission Controllers

- Admission controller에 `NamespaceAutoProvision` 옵션 추가
- `kube-apiserver -h | grep enable-admission-plugins` 명령어로 사용 가능한 admission controller 조회 가능

    - `kube-apiserver.service`에 `--enable-admission-plugins=NodeRestriction,NamespaceAutoProvision` 추가 
    - 같은 위치에 `--disable-admission-plugins=DefaultStorageClass` 추가


- kubeadm의 경우, `kubectl exec kube-apiserver-controlplane -n kube-system --kube-apiserver -h | grep enable-admission-plugins` 
    - `/etc/kubernetes/manifests/kube-apiserver.yaml`의 spec.containers/command에 `--enable-admission-plugins=NodeRestriction,NamespaceAutoProvision` 추가

- 해당 설정 후 blue라는 namespace가 없을때 `kubectl run nginx --image=nginx --namespace=blue`하면 blue라는 namespace 생성 후 pod 생성

- 현재는 NamespaceAutoProvision과 NamespaceExists가 사라지고 NamespaceLifecycle로 변경 > 존재하지 않는 namespace의 경우 아예 생성되지 않음

## Validating and Mutating Admission Controllers

#### Validating Admission Controller

- 직전에 본 NamespaceExists 등이 유효성 검사 후 허용 또는 거부하는 Validating Admission Controller

#### Mutating Admission Controllers

1. Mutating Admission Controller

- 생성되기 전에 개체 또는 요청 자체를 변경하거나 변경할 수 있음

- `DefaultStorageClass`는 PVC 생성시 storageClassName을 지정하지 않은 경우 default로 지정해주는 admission controller

- `NamespaceAutoProvision`

#### 자체 admission controller

1. webhook을 이용해 클러스터 내부나 외부에 호스팅된 서버를 가리킬 수 있음
- webhook을 호출하면 Admission Webhook Server 호출
- webhook server는 요청이 허용되는지 아닌지에 따라 허용 또는 거부함

2. Admission Controller 종류
- MutatingAdmission Webhook
- ValidatingAdmission Webhook

#### 자체 admission controller 설치 단계

1. Admission Webhook Server 배포

- Admission Webhook Server는 고유 로직 존재
- 배포
    - 어떤 플랫폼에서든 구축될 수 있는 apiserver가 될 수 있어 webhook server 코드는 Kubernetes 문서 페이지에서 확인 가능
    - 유일한 요구 사항은 mutate를 수락하고 API 유효성 검사하고 웹서버가 기대하는 JSON으로 반응 
    - admission webhook server 코드를 이해할 필요없고 해당 코드에는 `요청을 승인하거나 거부하는 로직이 있다`는 것만 이해하면 됨
    - webhook server를 컨테이너화해 Kubernetes cluster에 service(webhook-service)와 함께 배포

2. Webhook(클러스터) 구성

- service에 요청의 유효성을 검사하거나 변형시키기 위해 클러스터 구성

- `clientConfig`: webhook server를 등록하는 위치 구성. 
    - webhook server를 외부적으로 자체 배포하는 경우 해당 서버의 URL 경로 
    ```
    clientConfig:
        url: "https://external-server.example.com"
    ```
    - kubernetes에 배포한 경우, namespace와 service name 이용. apiserver와 webhook server 간의 통신은 TLS 위에 있어야 함
    ```
    clientConfig:
        service:
            namespace: "webhook-namespace"
            name: "webhook-service"
        caBundle: "[TLS]"
    ```

- `rules` : apiserver를 호출하는 경우

```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
    name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  clientConfig:
    service:
        namespace: "webhook-namespace"
        name: "webhook-service"
    caBundle: "Ci0tLS0tQk...tLS0K"
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
```


## API Versions

- 이전에 `/apis` 이야기할 때 아래에 순서대로 API Groups, Versions, Resources, Verbs 존재

- API group은 version으로 /v1 뿐 아니라 /v1alpha1, /v1beta1과 같은 버전 가질 수 있음
    - /v1이 GA(Genaral Available. 일반적으로 사용 가능한) 버전

1. Alpha

- API가 최초로 개발되어 쿠버네티스 부호에 병합된 후 최초로 출시된 쿠버네티스 버전
- 테스트가 부족하고 버그가 있을 수 있음
- 출시될때 사용가능하다는 보장 x

2. Beta

- Alpha 버전의 주요 버그를 모두 고치고 최종 테스트를 마치면 Beta 버전이 됨
- 종단 간 테스트
- GA가 아니므로 버그 존재할 수 있음
- 나중에 GA로 옮기기로 약속

3. GA(Stable)

- Beta 단계에서 몇 개월을 보낸 후 몇 번의 출시 후 API Group은 GA로 이동
- 신뢰도가 높음
- default

4. 버전이 여러 개인 경우
- API Group은 여러 버전을 동시에 지원 가능
    - ex. /apps의 버전은 /v1, /v1alpha1, /v1beta1 3개 존재
- nginx 만드는 yaml 파일의 apiVersion으로 apps/v1alpha1, apps/v1beta1, apps/v1 3개 모두 가능
- 단 `하나의 버전이 저장소 버전`이 될 수 있음 => GA
    - 어떤 개체가 저장소로 이동하면 저장소 버전으로 변경

5. Preferred Version

- url에 `127.0.0.1:8001/apis/[원하는 API Group명]`에서 조회 가능

6. Storage Version

- 주로 Preferred Version과 동일하지만 반드시 일치하지는 않음
- 특정 API의 저장소 버전을 볼 수 없음

7. Enabling/Disabling API groups

- 특정 버전을 활성화하거나 비활성화하려면 kube-apiserver의 runtime-config에 추가
- 수정 후 서비스 재실행해야함

## API Deprecations

1. API Deprecation Policy Rule #1

- API elements may only be removed by incrementing the version of the API group 
- v1alpha1은 v1alpha2가 존재하지 않는 이상 계속 존재

2. API Deprecation Policy Rule #2

- API objects must be able to round-trip between API versions in a given release without information loss, with the exception of whose REST resources that do not exist in some versions.


3. API Deprecation Policy Rule #4a

- Other than the most recent API versions in each track, older API versions must be supported after their announced deprecation for a duration of no less than:
    - GA: 12 months or 3 releases (whichever is longer)
    - Beata: 9 months or 3 releases (whichever is longer)
    - Alpha: 0 releases

- 반드시 해당 기간동안은 지원되어야함

4. API Deprecation Policy Rule #4b
- The "preferred" API version and the "storage version" for a given group may not advance until after a release has been made that supports both the new version and the previous version

- GA가 나오기 전에는 storage version이 변하지 않음 => 최초의 beta version

5. API Deprecation Policy Rule #3

- An API version in a given track may not be deprecated until a new API version at least as stable is released.

- v1alpha1이 나와도 기존의 GA인 v1을 제거하면 안됨
- alpha 버전은 GA 버전을 감소시키지 않음

6. Kubectl convert

- `kubectl convert -f [OLD FILE NAME] --output-version [NEW API VERSION]` 명령어로 버전 변환 가능
    - 해당 명령어는 plugin으로 별도로 설치해야함

## Custom Resource Definition

#### Resources

1. resource 생성시 resource 생성 후 그 정보를 etcd 데이터 저장소에 저장
2. kubernetes 내 `controller`가 내장되어있음
    - controller는 백그라운드에서 동작하는 프로세스로 관리해야하는 리소스의 상태를 지속적으로 모니터링
    - 지정한 replicas만큼 리소스 생성

#### Custom Resource

`flightticket.yaml`
```
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
    name: my-flight-ticket
spec:
    from: Mumbai
    to: London
    number: 2
```

#### Custom Resource Definition(CRD)

- 원하는 유형의 resource 생성 가능

- 위 Custom Resource를 생성할 때 apiVersion이 맞지 않다는 오류 발생 => 미리 만들어줘야함
- `spec.scope`: namespace 유무
- `spec.group` : API version이 제공되는 API grou
- `spec.names.kind`: Custom Resrource의 kind와 일치해야함
- `spec.names.singular`: 단수 이름
- `spec.names.plural`: 복수 이름
- `spec.names.shortNames` : 축약형
- `spec.versions.schema.openAPIV3Schema`: 어떤 영역을 지원하는지 또는 영역을 지원하는 값의 유형
    - Custom Resource의 spec에 해당하는 내용

`flightticket-custom-definition.yaml`
```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
    name: flighttickets.flights.com
spec:
    scope: Namespaced
    group: flifhts.com
    names:
        kind: FlightTicket
        singular: flightticket
        plural: flighttickets
        shortNames:
            - ft
    versions:
        - name: v1
          served: true
          storage: true
          schema:
            openAPIV3Schema:
                type: object
                properties:
                    spec:
                        type: object
                        properties:
                            from:
                                type: string
                            to:
                                type: string
                            number:
                                type: integer
                                minimum: 1
                                maximum: 10

```


## Custom Controllers

## Operator Framework

## Labs 실습

1. KubeConfig

Q1

- `ls -a`로 숨겨진 파일도 전체 조회해 kubeconfig 확인
- my-kube-config는 새로 생성한 파일

Q12

- 사용하려는 context가 기본 context file이 아닌 다른 파일에 존재하는 경우 `kubectl config --kubeconfig=[CONFIG FILE 경로] use-context [사용하려는 CONTEXT NAME]`

Q13

- Sol: `mv /root/my-kube-config /root/.kube/config`로 파일 이동

Q14

-Sol: `ls /etc/kubernetes/pki/users`에 사용할 수 있는 users 존재

2. RBAC

Q1

- `kubectl describe pod kube-apiserver-controlplane -n kube-system`로 kube-apiserver 조회 가능
- Sol: `cat /etc/kubernetes/manifests/kube-apiserver.yaml`에서 확인 가능

Q3

- namespace 전체 `k get roles -A`

Q8

- Sol: `k get pods --as dev-user`

3. ClusterRoles

Q1

- `k get clusterroles  --no-headers | wc -l`로 헤더 제외 숫자 세기

Q8

- Sol: `kubectl api-resources`에서 축약어 조회 가능

4. Admission Controllers

Q3

- `vim /etc/kubernetes/manifests/kube-apiserver.yaml` 조회
- Sol: 위와 같이 yaml 파일 들어간 후 `/[찾고자하는단어]` 작성하면 해당 파일에서의 해당 단어가 나타남
- Sol: 바로 `grep enable-admission-plugins /etc/kubernetes/manifests/kube-apiserver.yaml`도 가능

5. Validating and Mutating Admission Controllers

Q2

- Mutating과 Validating은 Mutating이 먼저 진행되고 Validating이 진행됨

Q11

- Sol: `kubectl get pod pod-with-defaults -o yaml`으로 생성된 pod의 yaml 파일에서 securityContext 조회

6. API Versions/Deprecations

Q1

- Sol: 축약어 보고싶은 경우 `kubectl api-resources`

Q2 

- Sol: `1.22.2`에서 첫번째는 major, 두번째는 minor, 세번째가 version

Q3

- Sol: `kubectl explain job`의 출력 version이 `batch/v1`인 경우 batch가 API Group, v1이 버전

Q4

- Sol: `kubectl proxy 8001&` 명령어 실행해 proxy server 백그라운드로 실행 > `curl localhost:8001/apis/authorization.k8s.io`

- `?: 8001은 어떻게 알 수 있지 ?` => custom port 번호

Q5

- Sol
    - 실패할 수 있으므로 파일 백업 `cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.backup`
    - 파일 수정 `vim /etc/kubernetes/manifests/kube-apiserver.yaml`에 `--runtime-config=rbac.authorization.k8s.io/v1alpha1`
        - `runtime-config`는 내장 API를 활성화하거나 비활성화
    - 적용하는데 시간 소요 => `kubectl get pod -n kube-system`에서 Pending 상태면 Running 될 때까지 기다려야함

Q7

- Sol: convert 설치(Q6) 후 > `kubectl convert -f [파일명] --output-version [바꾸려는 버전]` > `kubectl apply -f [파일명]`


7. 