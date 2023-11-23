## Define, build and modify container images

1. Image 필요성

- Docker Hub에서 application의 일부로 사용하고자하는 구성 요소나 서비스를 찾을 수 없기 때문

- 개발 중인 application의 편리한 운송과 배포를 위해

- 본 section에서 application을 container화할 것 => `Flask` 프레임워크와 Python 사용하여 만든 웹 application

2. Image 생성 방법

- 어떤 application을 container화하는지, 어떤 application을 위한 image를 만드는지, application이 어떻게 build되는지 이해해야함

- application 수동 배포 단계

  - 1. Ubuntu와 같은 OS
  - 2. apt 명령어로 (source) repository 업데이트
  - 3. apt 명령어로 install dependency 설치
  - 4. pip 명령어로 Python dependency 설치
  - 5. application의 source code를 opt 폴더와 같은 위치에 복사
  - 6. flask 명령어로 웹 서버 실행

- 위의 설명은 원하는 Image를 만드는 process의 개요 역할

- local image 생성 단계
  - 1. `Dockerfile`이라는 이름의 Docker file 생성
  - 2. application에 설정할 instruction 작성: Ex. dependency, source code를 가져올 위치, application의 entrypoint 등
  - 3. Docker build 명령을 이용해 image build. 이때 Dockerfile과 image에 대한 tag 이름 명시 `docker build Dockerfile -t mmumshad/my-custom-app`
  - 4. public Docker Hub repository에서 사용 가능하도록 `docker push mmumshad/my-custom-app` 명령어 실행

`Dockerfile`

```
FROM Ubuntu

RUN apt-get update
RUN apt-get install python

RUN pip install flask
RUN pip install flask-mysql

COPY . /opt/source-code

ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```

3. Dockerfile

- 특정 형식으로 작성된 텍스트 파일

- `INSTRUCTION + ARGUMENT` 형식

  - `INSTRUCTION`은 Dockerfile에서 왼쪽에 대문자로 되어있고 각각의 INSTRUCTION은 Docker image를 만드는 동안 특정 동작을 수행하도록 함. EX. FROM, RUN, COPY, ENTRYPOINT
  - `ARGUMENT`는 INSTRUCTION에 대한 인자

- 위의 Dockerfile 설명
  - `FROM Ubuntu`는 container의 기본 OS 설치하는 명령어. 모든 docker image는 다른 image(OS 또는 OS에 근거해 만들어진 다른 image)에 근거해야함. Docker Hub에서 모든 OS의 공식 릴리즈 Image를 찾을 수 있음. 모든 Dockerfile은 반드시 INSTRUCTION에서부터 시작해야함
  - `RUN` INSTRUCTION를 통해 image에 설치되어야 할 dependency를 설치
  - `COPY` INSTRUCTION은 docker image에 local system에 있는 파일로부터 source code를 복사
  - `ENTRYPOINT` INSTRUCTION은 image가 container로 실행될 떄 실행되는 명령어

4. Layered architecture

`Dockerfile`

```
FROM Ubuntu
RUN apt-get update && apt-get -y install python
RUN pip install flask flask-mysql
COPY . /opt/source-code
ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```

- docker build 명령어로 build시 layered architecture 안에 build됨: `docker build Dockerfile -t mmumshad/my-custom-app`
- 각각의 INSTRUCTION이 docker image에 새로운 layer 생성. 이전 layer에서 변경된 것만
  - Layer1. Base Ubuntu OS Layer
  - Layer2. Changes in apt packages
  - Layer3. Changes in pip packages
  - Layer4. Source code
  - Layer5. Update Entrypoint with "flask" command
- 각 layer는 이전 layer의 변화만 저장하기 때문에 크기도 반영 : `docker history [IMAGE NAME]` 명령어로 크기 조회 가능

- docker build 명령을 실행하면 관련된 다양한 단계와 각 작업의 결과 조회 가능
- Layered architecture는 docker build가 실패하는 경우 특정 단계부터 다시 시작하도록 도우며 build process에 새로운 단계를 추가하는 경우 처음부터 다시 시작할 필요가 없음

- docker fail의 경우
  - Layer3가 실패하여 문제를 해결한 후 다시 docker build를 재실행하는 경우 또는 docker file에 layer를 추가하는 경우, 변화되지 않은 layer는 cache에서 이전 layer를 재사용
- image 재구축이 빠르며 docker가 매번 전체 image를 재구축하길 기다릴 필요 X
- application의 source code가 자주 변할 수 있기에 유용

5. containerize 가능 도구

- database, devops tool, OS 뿐 아니라 browser와 같은 간단한 것, curl과 같은 utility, Spotify 또는 Skype와 같은 application 컨테이너화 가능
- 모든 것이 container화가 가능하므로 설치가 아닌 docker를 사용해 실행

## Commands and Arguments in Docker

1. Docker container

- Docker container에 Ubuntu 실행 `docker run ubuntu`

  - `docker ps`로 실행 중인 container를 조회하면 아무것도 나오지 않음
  - `docker ps -a`로 전체 container 조회시 Exited STATUS로 나타남
  - VM과 달리 container는 OS를 호스팅하도록 되어 있지 않음. container는 특정 작업이나 process를 실행하도록 만들어지고 작업이 끝나면 container가 종료됨. ex. seb server, application server, database instance, 연산이나 분석 등
  - 따라서 container는 내부의 프로세스가 동작 중이어야 동작

- container 내에서 실행되는 프로세스 정의
  - 인기 Docker image인 `nginx`에 대한 Docker file에는 `CMD`라는 INSTRUCTION 존재 `CMD ["nginx"]`
  - `CMD`란 프로그램을 정의하는 명령의 약자로, 시작되면 container 내에서 실행될 명령
  - 위에서 진행한 작업은 ubuntu OS로 container를 실행하는 것. 해당 이미지에 대한 docker file에는 `CMD ["bash"]`
  - `BASH`는 web server나 database server와 같은 프로세스가 아니라 터미널에서 input을 듣는 shell. 터미널을 찾지 못하는 경우 Exit
  - Docker가 default로 실행 중일 때 container에 terminal을 연결하지 않음. 따라서 container에 terminal에 연결하지 않으므로 Bash 프로그램은 terminal을 찾지 못해 Exit

2. continaer run command 지정

- `docker run [IMAGE] [COMMAND]` Docker run command에 추가 옵션 추가 => 이미지에 지정된 기본 명령 재정의

  - `docker run ubuntu sleep 5`

- Docker file에 `CMD [COMMAND] [PARAMETER]` 영구적으로 지정
  - Docker file 재정의
  - `docker build -t ubuntu-sleeper .` 로 image build
  - `docker run ubuntu-sleeper`

```
FROM Ubuntu
CMD sleep 5     //or CMD ["sleep","5"]
```

- `docker run` 실행 시 sleep 시간을 지정하고 싶은 경우

  - 안 좋은 예시: Docker file은 `FROM Ubuntu CMD sleep 5`, `docker run ubuntu-sleeper sleep 10`
  - Docker file은 `FROM Ubuntu ENTRYPOINT ["sleep"]`, `docker run ubuntu-sleeper 10`
    - 위의 예시와 다르게 ENTRYPOINT를 sleep으로 두어 docker run 실행 시 sleep을 또다시 작성할 필요 X
    - 하지만 이 경우 docker run ubuntu-sleeper 만 실행하는 경우 오류 발생 => default 지정이 안되어있기 때문
  - Docker file은 `FROM Ubuntu ENTRYPOINT ["sleep"] CMD ["5"]`, `docker run ubuntu-sleeper`의 경우 sleep 5 동작, `docker run ubuntu-sleeper 10`의 경우 sleep 10 동작

- runtime 동안 ENTRYPOINT를 수정하고 싶은 경우, `docker run --entrypoint sleep2.0 ubuntu-sleeper 10`을 하는 경우, `ENTRYPOINT ["sleep2.0"]`으로 수정되고 10초동안 동작

- 즉, 기본적으로 5초동안 sleep 상태인 docker image 생성. docker run 명령어를 통해 sleep 시간 재정의 가능

## Commands and Arguments in Kubernetes

1. docker image

- 위에서 생성한 `ubuntu-sleeper` image로 pod 생성
- `docker run --name ubuntu-sleeper ubuntu-sleeper` or `docker run --name ubuntu-sleeper ubuntu-sleeper 10`

2. yml 파일 생성

- 기본 : `docker run --name ubuntu-sleeper ubuntu-sleeper`

`pod-definition.yml`

```
apiVersion: v1
kind: Pod
metadata:
    name: ubuntu-sleeper-pod
spec:
    containers:
        - name: ubuntu-sleeper
          image: ubuntu-sleeper
```

- argument 변경 시 yaml : `docker run --name ubuntu-sleeper ubuntu-sleeper 10`의 경우 pod

  - `docker run` 명령어에 추가된 내용은 pod 생성 시 모두 Pod Definition File의 spec에 들어감
  - `pod-definition.yml`의 spec>containers에 `args: ["10"]` 추가
  - pod에 지정된 args는 docker file의 CMD를 무효화

- ENTRYPOINT 변경 : `docker run --name ubuntu-sleeper --entrypoint sleep2.0 ubuntu-sleeper 10`의 경우 pod
  - `pod-definition.yml`의 spec>containers에 `command: ["sleep2.0"]` 추가
  - pod에 지정된 command는 docker file의 ENTRYPOINT를 무효화

`Dockerfile`

```
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

`pod-definition.yml`

```
apiVersion: v1
kind: Pod
metadata:
    name: ubuntu-sleeper-pod
spec:
    containers:
        - name: ubuntu-sleeper
          image: ubuntu-sleeper
          command: ["sleep2.0"]
          args: ["10"]
```

|      Dockerfile      |         YAML          |
| :------------------: | :-------------------: |
| ENTRYPOINT ["sleep"] | command: ["sleep2.0"] |
|      CMD ["5"]       |     args: ["10"]      |

## A quick note on editing Pods and Deployments

1. Edit Pod

- Pod는 아래의 spec만을 편집할 수 있음

  - spec.containers[*].image
  - spec.initContainers[*].image
  - spec.activeDeadlineSeconds
  - spec.tolerations

- 이외의 running 상태인 pod의 environment variables, service accounts, resource limits를 수정할 수는 없지만 정말 원할 경우 가능한 두 가지 옵션 존재

2. 두 가지 옵션

- 1. `kubectl edit pod [POD NAME]` 명령어 실행해 vi 편집기에서 pod open. 원하는 spec 수정하고 저장 시 오류 발생 => 수정불가능한 편집기 사용했기 때문
  - 저장 시 오류가 발생하지만 temporary location에 수정한 파일의 복사본 저장
  - 기존의 pod 삭제 후 temporary location에 존재하는 file로 pod 생성
  - `kubectl delete pod webapp` > `kubectl create -f /tmp/kubectl-edit-ccvrq.yaml`

![Alt text](image-1.png)

- 2. `kubectl get pod webapp -o yaml > my-new-pod.yaml` 명령어를 통해 pod definition file을 YAML로 추출
  - vi 편집기로 파일 변경 `vi my-new-pod.yaml`
  - 기존의 pod 삭제 후 변경한 파일로 pod 생성
  - `kubectl delete pod webapp` > `kubectl create -f my-new-pod.yaml`

3. Edit Deployment

- Deployments 사용해 pod의 field와 spec 쉽게 편집 가능
- pod는 deployment의 하위개념이므로 deployment 변경 시 자동으로 기존 pod는 삭제되고 새로운 pod 생성
- `kubectl edit deployment my-deployment`

## Environment Variables

1. Environment Variables

- `docker run -e APP_COLOR=pink simple-webapp-color`

- pod definition file
  - `env` property 사용. env는 array로, env 속성 하의 모든 항목은 배열 내의 항목을 나타내는 대시(-)부터 시작
  - env 하위의 name은 환경변수의 이름으로 container와 함께 사용 가능, value는 해당 환경변수의 값
  - 아래의 yaml 파일의 environment value는 `key-value` 포맷을 이용하여 환경변수 직접 지정

`pod-definition.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      env:
        - name: APP_COLOR
          value: pink
```

2. ENV Value Type

- Plain Key Value

```
env:
  - name: APP_COLOR
    value: pink
```

- ConfigMaps

```
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
```

- Secrets

```
env:
  - name: APP_COLOR
    valueFrom:
      secretKeyRef:
```

- key-value와 configmap,secret의 차이점은 값을 지정하는 대신값을 지정하고 configmap이나 secret의 사양에 따른다는 점

## ConfigMaps

- 이전 강의에서 pod definition file에서 environment variable 정의하는 법 보여줌 => pod definition file이 많은 경우 Query file에 저장된 environment data를 관리하기 어려움

1. ConfigMap

- ConfigMap을 사용하면 pod definition file에서 environment variable 정보를 가져와 중앙에서 관리 가능

- ConfigMap은 Kubernetes의 key-value 쌍으로 configuration data를 전달하는데 사용
- pod가 생성되면 pod에 configmap을 삽입해 key-value 쌍이 environment value로 사용되어 pod의 container 안에서 host된 application program을 위해 동작

- 기존 pod-definition
  `pod-definition.yaml`

```
...
spec:
  containers:
      ...
      env:
        - name: APP_COLOR
          value: blue
        - name: APP_MODE
          value: prod
```

- ConfigMap 사용
  `ConfigMap`

```
APP_COLOR: blue
APP_MODE: prod
```

`pod-definition.yaml`

```
...
spec:
  containers:
      ...
      envFrom:
        - configMapRef:
            name: app-config
```

2. ConfigMap 구성

- STEP1: Create ConfigMap: Imperative

  - Imperative: `kubectl create configmap`. ConfigMap definition file 생성 X
  - 1. 명령어에서 `--from-literal`를 이용해 명령 자체에서 key-value 쌍 직접 지정
    - `kubectl create configmap [CONFIG-NAME] --from-literal=[KEY]=[VALUE]`
    - `kubectl create configmap app-config --from-literal=APP_COLOR=blue`
    - 둘 이상의 config를 지정하고 싶은 경우 `--from-literal` 옵션 여러 번 지정
  - 2. 명령어에서 `--from-file`을 이용해 파일을 통해 config data 지정
    - `kubectl create configmap [CONFIG-NAME] --from-file=[PATH-TO-FILE]`
    - `kubectl create configmap app-config --from-file=app_config.properties`

- STEP1: Create ConfigMap: Declarative

  - Declarative: `kubectl create -f`. ConfigMap definition file 생성
  - `config-map.yaml` 이름의 definition file 생성. Pod의 definition과 달리 `spec 대신 data` field 존재
    - data field 아래에 key-value 형식으로 config data 추가

  `config-map.yaml`

  ```
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-config
  data:
    APP_COLOR: blue
    APP_MODE: prod
  ```

  - configmap은 나중에 pod와 관련해 사용하기에 이름을 적절히 붙여줘야함

  `app-config`

  ```
  APP_COLOR: blue
  APP_MODE: prod
  ```

  `mysql-config`

  ```
  port: 3306
  max_allowed_packet: 128M
  ```

  `redis-config`

  ```
  port: 6379
  rdb-compression: yes
  ```

- STEP2: Inject into Pod

  - 1. `ENV` : Pod definition file에 `envFrom` property 추가
  - `envFrom`은 리스트 형태로, 필요한 만큼 environment variable을 넘길 수 있으며 list의 각 항목은 configmap과 일치

  `pod-definition.yaml`

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: simple-webapp-color
    labels:
      name: simple-webapp-color
  spec:
    containers:
      - name: simple-webapp-color
        image: simple-webapp-color
        ports:
          - containerPort: 8080
        envFrom:
          - configMapRef:
              name: config-map  //configmap 이름
  ```

  - `kubectl create -f pod-definition.yaml`을 통해 configmap에 나타난 blue 배경으로 만들어진 웹 프로그램 생성
  - 2. `SINGLE ENV`: 단일 환경 변수

  ```
  env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
  ```

  - 3. `VOLUME`: 전체 데이터를 파일로 volume에 넣을 수 있음

  ```
  volumes:
    - name: app-config-volume
      configMap:
        name: app-config
  ```

3. 명령어

- `kubectl get configmaps` : configmap 조회
- `kubectl describe configmaps`: config data를 data 섹션 아래에 나열

## Secrets

1. Secrets

- 민감한 정보를 저장하는데 쓰임. ex.password, key
- configmap과 비슷하지만 인코딩된 포맷에 저장되어있음

2. Secrets 구성

- STEP1. Create Secret : Imperative

  - Imperative: `kubectl create secret generic`. Secret definition file 생성 X
  - 1. 명령어에서 `--from-literal`를 이용해 명령 자체에서 key-value 쌍 직접 지정
    - `kubectl create secret generic [SECRET-NAME] --from-literal=[KEY]=[VALUE]`
    - `kubectl create secret generic app-secret --from-literal=DB_HOST=mysql --from-literal=DB_User=root --from-literal=DB_Password=paswrd`
    - 둘 이상의 config를 지정하고 싶은 경우 `--from-literal` 옵션 여러 번 지정
  - 2. 명령어에서 `--from-file`을 이용해 파일을 통해 secret data 지정
    - `kubectl create secret generic [SECRET-NAME] --from-file=[PATH-TO-FILE]`
    - `kubectl create secret generic app-secret --from-file=app_secret.properties`

- STEP1. Create Secret : Declarative
  - Declarative: `kubectl create -f`. Secret definition file 생성
  - yaml에 저장하는 데이터를 인코딩된 형식으로 데이터 지정
  - 인코딩 값은 `echo -n 'mysql' | base64`와 같이 원하는 데이터를 넣어 출력
  - 인코딩한 데이터를 디코딩하고 싶은 경우 `echo -n 'bXlzcWw=' | base64 --decode`와 같이 디코딩하고 싶은 데이터를 넣어 출력

`secret-data.yaml`

```
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=       #mysql
  DB_User: cm9vdA==       #root
  DB_Password: cGFzd3Jk   #paswrd
```

- STEP2. Inject into Pod
  - 1. `ENV` : Pod definition file에 `envFrom` property 추가
  - `envFrom`은 리스트 형태로, 필요한 만큼 environment variable을 넘길 수 있으며 list의 각 항목은 secret과 일치

`pod-definition.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      envFrom:
        - secretRef:
            name: app-secret
```

- 2. `SINGLE ENV`

```
env:
  - name: DB_Password
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_Password
```

- 3. `VOLUME`
  - `ls /opt/app-secret-volumes` 명령어로 존재하는 Secret key 이름의 파일 조회 가능
  - `cat /opt/app-secret-volumes/DB_Password` 명령어로 key에 대한 value 조회 가능

```
volumes:
  - name: app-secret-volume
    secret:
      secretName: app-secret
```

3. Note on Secrets

- Secrets are not Encrypted. Only encoded

  - secret은 암호화되어있지 않고 인코딩되어 있기에 누구나 파일을 조회할 수 있고 secret 조회 가능
  - Do not check-in Secret objects to SCM along with code.

- Secrets are not encrypted in ETCD
  - Enable encryption at rest
- Anyone able to create pods/deployments in the same namespace can access the secrets

  - Configure least-privilege access to Secrets - `RBAC` : RBAC을 구성해 액세스 제한 고려

- Consider `third-party secrets store providers` AWS Provider, Azure Provider, GCP Provider, Vault Provider : secret은 etcd가 아닌 외부 secret store provider에 저장되고 provider는 보안의 대부분을 처리

## A quick note about Secrets

- secret은 데이터를 base64 형식으로 인코딩해 저장하여 누구나 쉽게 해독 가능
  <https://kubernetes.io/docs/concepts/configuration/secret/>

- secret 모범 사례

  - secret definition file을 source code repository에 체크인하지 않음
  - 암호화된 상태로 ETCD에 저장되도록 암호화 활성화
    <https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/>

- Kubernetes의 Secret 사용
  - secret은 해당 노드의 pod가 필요로하는 경우에만 노드로 전송
  - Kubelet은 secret을 tmpfs에 저장하여 disk 저장소에 기록되지 않도록 함
  - secret에 의존하는 Pod가 삭제되면 kubelet은 secret data의 로컬 복사본도 삭제

<https://kubernetes.io/docs/concepts/configuration/secret/#protections>
<https://kubernetes.io/docs/concepts/configuration/secret/#risks>

- Kubernetes에서 민감한 데이터를 처리하는데 Helm Secrets, HashiCorp Vault와 같은 도구를 사용하는 것도 하나의 방법

## Demo: Encrypting Secret Data at Rest

<https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/>

1. Secret

- `kubectl create secret generic my-secret --from-literal=key1=supersecret`
- `kubectl describe secret my-secret`
- `kubectl get secret my-secret -o yaml` : 인코딩된 secret value 확인 가능
- `echo "[인코딩된 값]" | base64 --decode` : 인코딩된 값을 디코딩해 실제 값 확인 가능

2. etcd 서버에 데이터가 저장되는 법

- 1. etcdctl 설치

  - `apt-get install etcd-client`. etcd가 없는 경우 etcd server는 pod 안에서 실행. 클라이언트만 서버에서 여전히 포드로 실행 중
  - `kubectl get pods -n kube-system`을 통해 etcd-controlplane 확인

- 2. 아래 코드 실행
  - `ls /etc/kubernetes/pki/etcd/`에 ca.crt, server.crt, server.key 존재하는지 확인
  - ca.crt는 인증서 파일로 etcd server에 연결 또는 인증을 위해 필요
  - 아래 코드는 etcd api version을 3으로 설정. 마지막에 16진수 추가

```
ETCDCTL_API=3 etcdctl \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key  \
   get /registry/secrets/default/[SECRET NAME] | hexdump -C
```

- 3. 결과
  - etcd에는 아래의 형태로 데이터 저장
  - 암호화되지 않은 포맷으로 저장 => 누구나 접근하여 secret 값 조회 가능하다는 문제 발생

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 73 65 63 72 65 74  |s/default/secret|
00000020  31 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |1.k8s:enc:aescbc|
00000030  3a 76 31 3a 6b 65 79 31  3a c7 6c e7 d3 09 bc 06  |:v1:key1:.l.....|
00000040  25 51 91 e4 e0 6c e5 b1  4d 7a 8b 3d b9 c2 7c 6e  |%Q...l..Mz.=..|n|
00000050  b4 79 df 05 28 ae 0d 8e  5f 35 13 2c c0 18 99 3e  |.y..(..._5.,...>|
[...]
00000110  23 3a 0d fc 28 ca 48 2d  6b 2d 46 cc 72 0b 70 4c  |#:..(.H-k-F.r.pL|
00000120  a5 fc 35 43 12 4e 60 ef  bf 6f fe cf df 0b ad 1f  |..5C.N`..o......|
00000130  82 c4 88 53 02 da 3e 66  ff 0a                    |...S..>f..|
0000013a
```

3. Enable encryption

- STEP1. 암호화 주소가 이미 활성화되었는지 아닌지 판단 => `--encryption-provider-config` 속성 이용

  - `ps -aux | grep kube-api | grep "encryption-provider-config"`로 암호화된 REST가 이미 활성화되어있는지 확인
  - `ls /etc/kubernetes/manifests/`, `cat /etc/kubernetes/manifests/kube-apiserver.yaml`에서 "encryption-provider-config"이 없는 경우 Encryption이 활성화되지 않았다는 것 의미

- STEP2. configuration file 생성
  - encryption을 활성화하기 위해 기본적으로 필요한 단계로, configuration file을 만들어 특정 옵션으로 넘김
  - resources>providers의 파라미터들은 모두 다른 provider로 암호화 알고리즘으로 암호화한 방식이 다름 
  - provider의 순서가 중요한 이유는 첫번째 provider가 암호화에 사용됨 

```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:  #어떤 리소스를 암호화할지
      - secrets
    providers:  #provider를 이용해 암호화
      - identity: {}  #id 공급자는 암호가 전혀 없다는 뜻
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - secretbox:
          keys:
            - name: key1
              secret: YWJjZGVmZ2hpamtsb...
```

provider 참고: <https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/>

- 예시
  - 1. `head -c 32 /dev/urandom | base64` 값을 아래 파일의 [BASE 64 ENCODED SECRET]에 넣기
  - 2. 아래 `enc.yaml`을 volume으로 사용할 것이기에 파일 경로 변경: `mkdir /etc/kubernetes/enc` 폴더 생성 > `mv enc.yaml /etc/kubernetes/enc/` 생성한 폴더에 본 파일 이동

  `enc.yaml`
  ```
  apiVersion: apiserver.config.k8s.io/v1
  kind: EncryptionConfiguration
  resources:
    - resources:  #어떤 리소스를 암호화할지
        - secrets
      providers:  #provider를 이용해 암호화
        - aescbc:
            keys:
              - name: key1
                secret: [BASE 64 ENCODED SECRET]
        - identity: {}  #id 공급자는 암호가 전혀 없다는 뜻

  ```

  - 3. kube-apiserver의 manifest 파일 수정: `vi /etc/kubernetes/manifests/kube-apiserver.yaml`으로 파일 열기
    - `--encryption-provider-config=/etc/kubernetes/enc/enc.yaml` 추가. encryption configuration file이 어디에 있는지 알려줌
    - volumeMounts에 아래 코드 추가. 로컬 디렉토리는 이 경로에 마운트
    - volumes에 아래 코드 추가. 마운트 아래에 내 로컬 디렉토리의 위치를 지정할 것
    - 즉, volumes와 volumeMounts가 매핑되고 command를 통해 해당 yaml 파일(enc.yaml)을 사용할 수 있게 됨

  `kube-apiserver.yaml`
  ```
  ...
  spec:
    containers:
      - command:
          ...
          --encryption-provider-config=/etc/kubernetes/enc/enc.yaml # add this line
        volumeMounts:
          - name: enc                           # add this line
            mountPath: /etc/kubernetes/enc      # add this line
            readOnly: true                      # add this line
    volumes:
      - name: enc                             # add this line
        hostPath:                             # add this line
        path: /etc/kubernetes/enc           # add this line
        type: DirectoryOrCreate             # add this line
  ```

  - 4. kube-apiserver가 변경되었으므로 재시작 > `kubectl get pods`로 확인 또는 `crictl pods`로 확인. 
  - 5. `ps aux | grep kube-api | grep encry` 명령어를 통해 provider 지정 확인
  - 6. 새로운 Secret object 생성: `kubectl create secrets generic my-secret-2 --from-literal=key2=topsecret`
  - 7. 아래 명령어에서 SECRET NAME을 my-secret-2로 수정 후 실행하면 앞에서는 key2에 대한 value가 나타났지만 , encryption되었기에 이제는 보이지 않음
    - 기존의 my-secret은 value가 보임. encryption이 활성화된 이후에 생성된 secret만 encryption되기 때문
    - 기존의 secret도 모두 encryption 하고 싶은 경우 `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`

  ```
  ETCDCTL_API=3 etcdctl \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key  \
   get /registry/secrets/default/[SECRET-NAME] | hexdump -C
  ```

## Docker Security

1. Docker Security

- (Docker가 설치된) Host는 자신의 process 집합을 가지고 있어 다수의 OS process를 실행할 수 있음(Dockerd 또는 ESET 서버 등에서)

- `Process Isolation`
  - `docker run ubuntu sleep 3600`으로 1시간동안 sleep 상태인 Ubuntu Docker container 실행. 이떄 VM과 달리 container는 Host에서 완전히 격리되지않음.  
  - container와 Host는 같은 kernel 공유하고 contianer는 Linux의 namespace를 이용해 격리됨
  - Host는 Host만의 namespace를 가지고, container는 container만의 namespace를 가짐
  - container에 의해 실행되는 모든 process는 Host 내에서 실행되지만, 자신만의 namespace에서 실행됨
  - docker container 내에서 `ps aux` 하는 경우 하나의 프로세스만 보이고, Host에서 `ps aux` 하는 경우 sleep process를 포함하는 프로세스 리스트 조회 가능. process가 다른 namespace에서 다른 processId를 가질 수 있음 => Container Isolation(=Process Isolation)

2. Security - Users

- 1. container 내의 프로세스가 root user로 실행되는 것을 원치 않으면 `docker run` 명령어 실행 시 user option에 원하는 userId를 넣어 설정: `docker run --user=1000 ubuntu sleep 3600`
  - 이후 `ps aux`하면 USER가 1000으로 되어있음

- 2. User Security를 위한 다른 방법은 docker image 생성 시 정의
  - 이후 `docker run -t my-ubuntu-image .` > `docker run my-ubuntu-image sleep 3600`
  - `ps aux`하면 USER로 1000 확인 가능

`Dockerfile`
```
FROM ubuntu
USER 1000
```

3. Linux Capability

- Docker는 container 내 root 사용자의 능력 제한하는 보안 기능 집합 구현. container 내의 root 사용자는 Host의 root 사용자와 다름

- root 사용자는 시스템에서 가장 강력한 사용자. `/usr/include/linux/capability.h`에서 root 사용자가 접근할 수 있는 모든 리스트 조회 가능. 사용자가 사용할 수 있는 기능 제어 가능

- 기본적으로 Docker는 container를 실행할 때 기능 모음이 제한되어있어 container 내에서 실행되는 프로세스는 호스트를 재부팅하거나 같은 호스트에서 실행되는 호스트나 다른 컨테이너를 방해할 수 없음
  - 동작을 재정의하거나 다른 사용 가능한 기능보다 추가적인 특권을 제공하고 싶다면 `--cap-add` 옵션 사용
  - `docker run --cap-add MAC_ADMIN ubuntu`를 통해 MAC_ADMIN 기능 추가
  - `--cap-drop` 옵션으로 권한 포기 가능
  - `docker run --cap-drop KILL ubuntu`
  - `--privileged` 옵션으로 모든 권한 활성화 가능
  - `docker run --privileged ubuntu`

## Security Contexts

1. Container(Docker) Security

- docker를 실행할 때 보안 표준 집합을 정의하는 옵션 존재: `--user=[USERID]`, `--cap-add`, `--cap-drop`

2. Kubernetes Security

- Kubernetes에서는 container level 또는 pod level에서 보안 설정 가능
- pod와 container level에서 모두 적용되는 경우, container 설정이 pod 설정을 무효화함


- Pod level

```
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:  #Pod
    runAsUser: 1000
  containers
    - name: ubuntu
      image: ubuntu
      command: ["sleep","3600"]
```

- Container level
  - capabilities는 Container level에서만 지원하고 Pod level에서는 지원하지 않음

```
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  
  containers
    - name: ubuntu
      image: ubuntu
      command: ["sleep","3600"]
      securityContext:  #Container
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN]
```

## Service Account

1. Kubernetes Account

- User Account
  - 관리 작업을 수행하기 위해 클러스터를 액세스하는 Admin
  - Appliction 배포 등을 위해 클러스터에 액세스하는 Developer

- Service Account
  - application이 kubernetes 클러스터와 상호작용할 때 사용하는 계정. 
  - 모든 namespace에는 `default serviceaccount` 존재해 pod가 생성될 때마다 default service account와 token이 volumeMounts로 자동으로 해당 pod에 mount
    - 아래 파일을 통해 생성된 pod를 자세히 보면(`kubectl describe`), Mounts에 기본 설정된 secret-token 위치가 보이고, volumes에는 default-token 존재  
    - `kubectl exec -it my-kubernetes-dashboard --ls /var/run/secrets/kubernetes.io/serviceaccount` 하면 ca.crt, namespace, token 세 파일 존재
    - pod에 serviceAccount를 지정하고 싶은 경우 `serviceAccountName` 필드 사용
    - <span style="color:indianred">pod의 경우 기존 pod 삭제 후 다시 생성해야하는데, deployment의 경우 변화가 생기면 자동으로 롤아웃하므로 다시 생성할 필요 X</span>
    - Kubernetes는 아무것도 지정하지 않으면 자동으로 default service account 마운트. serviceaccount를 자동으로 마운트하지 않고 싶으면 `automountServiceAccountToken: false`

  `pod-definition.yml`
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-kubernetes-dashboard
  spec:
    containers:
      - name: my-kubernetes-dashboard
        image: my-kubernetes-dashboard
        #serviceaccount 지정
        serviceAccountName: dashboard-sa
  ```

  

  - ex1. Prometheus: 모니터링 application. 서비스 계정을 이용해 Kubernetes API를 퍼포먼스 매트릭으로 끌어옴
  - ex2. Jenkins: 자동화된 빌드 툴. 서비스 계정을 이용해 Kubernetes Cluster에 application 배포

- 예시: My Kubernetes Dashboard 구축
  - 파이썬에서 기본으로 제공되는 간단한 application
  - 배포할 때 하는 일은 Kubernetes API에 요청을 해서 Kubernetes Cluster의 Pod 목록 가져와 웹페이지에 표시
  - application이 Kubernetes API를 쿼리하려면 `Authentication` 필요. 이때 서비스 계정 이용

- Service Account 과정
  - 1. Service Account object생성: `kubectl create serviceaccount dashboard-sa`
  - 2. 생성 후 Token이 자동으로 생성됨: `kubectl describe serviceaccount dashboard-sa`로 확인 가능
    - token 생성 > secret object 생성 > secret 안에 token 저장
    - 해당 secret object는 service account에 링크됨
    - `kubectl describe secret [TOKEN]`으로 토큰 확인 가능. TOKEN은 service account describe 시 나타나는 이름
  - 3. token 사용 
    - curl을 이용해 인증 헤더로 전달자 토큰 제공 가능: `kubectl https://192.168.56.70:6443/api -insecure --header "Authorization: Bearer [REAL TOKEN]"`
    - My Kubernetes Dashboard에 토큰 넣어 인증

- Service Account: `Service Account 생성` > `RBAC`으로 권한 할당(CKA에서) > service account를 내보내 Kubernetes API에 대한 인증을 위한 `application 구성`에 사용

- application이 kubernetes cluster에 host 되어 있다면 hosting하는 pod 내부에 service token secret을 volume으로 자동으로 mount함으로써 간단하게 생성

2. v1.22/1.24 Update

- v1.22: Bound Service Account Tokens
  - `kubectl describe pod [POD NAME]`에서 serviceaccount 경로 찾기: `Mounts`
  - `kubectl exec -it [pod] ls [SERVICEACCOUNT 경로]` 실행해 세 가지 파일 확인

  - `jq -R 'split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson' <<< [REAL TOKEN]` 명령어로 토큰을 해독하거나 토큰을 JWT 웹사이트에 복사해 붙여넣을 수 있음

  - `jwt.io`의 PAYLOAD를 확인하면 만료일이 없음. 따라서 JWT 구현은 청중이나 시간에 구속되지 않음. serviceaccount가 존재하는 한 JWT는 유효하며 각 JWT는 서비스 당 개별 secret object 요구해 확장성 문제 초래

- v1.22: TokenRequestAPI
  - serviceaccount token의 안전성과 확장성을 높여주는 매커니즘 
  - serviceaccount token을 provision하는데 사용하며 Secret 대신 사용
  - API에서 생성된 토큰은 Audience Bound, Time Bound, Object Bound되어 더 안전
  - 새 pod가 생성되면 더이상 serviceaccount에 의존하지 않음 => TokenRequestAPI를 통해 정해진 수명을 가진 token 생성. pod 생성 시 이 token이 projected volume으로 투영되어 pod에 mount됨

`kubectl get pod my-kubernetes-dashboard -o yaml`
```
...
spec:
  containers:
    ...
    - volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-6mtg8
          readOnly: true
  volumes:
    - name: kube-api-access-6mtg8
      projected:
        defaultMode: 420
        sources:
          - serviceAccountToken:
              expirationSeconds: 3607
              path: token
  ...
```

- v1.24: Reduction of Secret-based Service Account Tokens
  - 과거에는 serviceaccount가 생성되면 만료되지 않는 토큰과 함께 자동으로 secret 생성. serviceaccount를 사용하는 pod에 자동으로 볼륨으로 마운트 => 현재는 serviceaccount만 생성하면 secret과 token이 생성 X. `secret은 아예 필요X`
  - pod에 자동으로 secret을 올리는 것이 변경되었고 대신 TokenRequest API로 이동
  📌 `kubectl create serviceaccount dashboard-sa`로 serviceaccount 생성 후 `kubectl create token dashboard-sa`로 내부에 토큰 생성하는 명령어 실행해야함
  - `jq -R 'split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson' <<< [REAL TOKEN]`에서 "exp"를 확인하여 토큰 유효 기간 확인 가능. 옵션을 통해 유효기간 설정 가능
  - secret을 만들고 싶은 경우, serviceaccount 생성 후 아래 yaml 실행해 secret 생성

  `secret-definition.yml`
  ```
  apiVersion: v1
  kind: Secret
  type: kubernetes.io/service-account-token
  metadata:
    name: mysecretname
    annotations:
      kubernetes.io/service-account.name: dashboard-sa
  ```

## Resource Requirements

1. 3개의 node 살펴보기

- 각 node에는 사용가능한 CPU와 Memory resource 세트 존재
- 하나의 pod가 CPU 2개와 Memory unit 1개가 필요하다고 가정. pod가 node에 놓일때마다 그 node에 있는 리소스 소비
- kube-scheduler는 pod에 요구되는 리소스 양을 고려하여 pod가 갈 node 결정
  - node에서 사용가능한 리소스가 모두 차면 pod 억제 : Pending 상태
- 각 pod에 필요한 리소스 요건
  - pod를 만들 때 필요한 CPU와 Memory 양 지정 가능. container에서 요청한 최소 resource 

`pod-definition.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      #resource 요건
      resources:
        requests:
          memory: "4Gi"
          cpu: 2
```

2. CPU

- 1 CPU는 100m(mili). 1m까지 설정할 수 있지만 그 이하는 안 됨
- 1 CPU = 1 AWS vCPU = 1 GCP Core = 1 Azure Core = 1 Hyperthread

3. Memory

- 1 Memory는 256 Mi(mebibyte)
- 1 Memory = 256Mi = 268435456 = 268M

- Gigabyte와 Gibibyte 비교

|단위|bytes|단위|bytes|
|:--:|:--:|:--:|:--:|
|1 G(Gigabyte)|1,000,000,000 bytes|1 Gi(Gibibyte)|1,073,741,824 bytes|
|1 M(Megabyte)|1,000,000 bytes|1 Mi(Mebibyte)|1,048,576 bytes|
|1 K(Kilobyte)|1,000 bytes|1 Ki(Kibibyte)|1,024 bytes|

4. Resource Limits

- Pod의 resource limit 가능
  - Ex. container에 vCPU 한 대만 사용하도록 제한, Memory를 512 Mi만 사용하도록 제한

  `pod-definition.yaml`
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: simple-webapp-color
    labels:
      name: simple-webapp-color
  spec:
    containers:
      - name: simple-webapp-color
        image: simple-webapp-color
        ports:
          - containerPort: 8080
        #resource 요건
        resources:
          requests:
            memory: "1Gi"
            cpu: 1
          limits:
            memory: "2Gi"
            cpu: 2
  ```

  - 위 pod가 생성되자 Kubernetes는 container의 새로운 limits 결정
  - pod 내 container 마다 requests와 limits 설정 => container가 여러 개면 각 container는 개별적으로 requests와 limits 설정 가능

5. Exceed Limits

- pod가 지정된 limit 이상의 자원을 사용하는 경우, CPU는 시스템이 CPU를 조절해 지정된 한도를 넘지 않도록하여 container는 제한 시간 이상 CPU 리소스를 사용할 수 없음

- Memory의 경우, container는 한도보다 많은 Memory 리소스 사용 가능. 계속해서 Memory를 소모하다가 container가 terminate됨 => 출력 결과에서 pod가 `OOM(Out Of Memory)` 오류로 종료

6. Default Behavior

- 기본적으로 Kubernetes는 CPU와 Memory requests와 limits가 없음 => 어떤 pod든 어떤 node에서 요구되는 만큼의 자원을 소비하고 리소스 노드에서 실행 중인 다른 pod나 process를 질식시킬 수 있음

- CPU: 하나의 Node에 pod가 2개 있는 경우 가정
  - No Requests & No Limits: limit이 없으므로 첫번째 pod가 노드의 모든 CPU 리소스를 소비하고, 두번째 pod의 리소스도 막을 수 있음 => 이상적이지 않음
  - No Requests & Limits: requests와 limits이 동일하게 3인 경우
  - Requests & Limits: 각 pod에는 vCPU 하나의 requests 보장. limit인 3개까지 가능하지만 그 이상은 불가능. 하지만 1번째 pod가 CPU 사이클이 더 필요하고, 2번째 pod가 CPU 사이클을 그만큼 소비하지 않는다면 1번째 pod의 CPU에 제한을 두고싶지 않음
  - Requests & No Limits: requests가 정해져있어 그만큼 vCPU를 보장받지만 limit이 없기에 pod마다 CPU 사이클을 최대한 많이 사용할 수 있음. 하지만 2번째 pod가 보장되는 CPU를 요청하면 해당 CPU 사이클은 보장 => 가장 이상적
    - 리소스를 제한하고 싶은 경우 제한 가능
    - Ex. Udemy에서 지원하는 kloud 환경. 클러스터에 컨테이너로 호스팅. 사용자가 원하는 모든 종류의 작업을 실행할 수 있으며 사용자가 인프라 오용을 막기 위해 limit 설정
    - application이 추가 CPU를 사용하도록 제한하고 싶지 않다면 limit을 설정하지 않아도 좋지만, 이때 모든 pod의 request 설정이 되어있어야함

- Memory
  - No Requests & No Limits: limit이 없으므로 두 pod는 경쟁 상태. 첫번째 pod가 node의 memory 전체를 소비하고 두번째 pod의 리소스를 막을 수 있음
  - No Requests & Limits: container는 자동으로 requests=limits으로 설정
  - Requests & Limits: 각 pod마다 memory 보장
  - Requests & No Limits: limit이 없기에 자신의 requests만큼은 보장받지만 더 필요한 경우 사용 가능한 메모리를 소모할 수 있음. 단 pod2가 pod1의 memory를 원하는 경우 kill. CPU와 달리 memory는 조절할 수 없음 => pod를 kill해서 memory를 풀고 자신이 그 memory 사용

7. LimitRange

- LimitRange
- 어떻게 모든 pod가 기본 설정으로 되어있는지 => LimitRange에서 가능
  - 기본 값을 정의하는데 도움을 줌. pod 내의 container에 대한 설정
  - pod definition file에 requests나 특정 limit 없이 생성된 contianer
  - namespace 내에서 가능
  - LimitRange 설정 전의 pod는 영향을 받지 않고, LimitRange 설정 후의 pod만 영향 받음

`limit-range-cpu.yaml`
```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
    - default:  #limit
        cpu: 500m
      defaultRequest: #request
        cpu: 500m
      max:  #limit
        cpu: "1"
      min:  #request
        cpu: 100m
      type: Container
```

`limit-range-memory.yaml`
```
apiVersion: v1
kind: LimitRange
metadata:
  name: memory-resource-constraint
spec:
  limits:
    - default:  #limit
        memory: 1Gi
      defaultRequest: #request
        memory: 1Gi
      max:  #limit
        memory: 1Gi
      min:  #request
        memory: 500Mi
      type: Container
```

8. Resource Quotas

- Kubernetes cluster에 배포된 앱이 사용할 수 있는 전체 resource를 제한하는 방법: Resource Quotas

- namespace level object로, requests와 limits을 엄격히 제한

`resource-quota.yaml`
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-resource-quota
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 10
    limits.memory: 10Gi
```

## Taints and Tolerations

- Pod와 Node의 관계: 어떤 Node에 어떤 Pod를 배치할지, 어떻게 제한할지

1. Taints와 Tolerations

- Taints와 Tolerations는 보안이나 침입과는 상관없고, 하나의 Node에 어떤 Pod로 스케쥴링할 수 있는지 제한을 설정하기 위해 사용

2. 예시

- Worker node가 3개인 간단한 cluster: Node 1,2,3과 Pod A,B,C,D가 존재한다고 가정
- 일반적인 상황
  - Pod가 생성되면 Kubernetes scheduler는 해당 pod를 node에 배치하려고함. restrict(제한)이나 limit이 없기 때문에 모든 node에 pod 배치
- 특정 사용 사례나 application에 대해 node1의 전용 리소스가 있다고 가정
  - 이 application에 속한 pod만 node1에 배치하길 원함
  - 1. 모든 pod가 node에 놓이는 것 방지. node에 `taint` 놓기: `Taint=blue`
    - 기본 설정상 pod는 구체적인 명시가 없으면 어떤 오류도 견딜 수 없음
    - 현재까지는 어떤 pod도 node1에 배치될 수 없음
  - 2. 특정 pod를 node1에 배치
    - pod D를 node1에 배치하고 싶은 경우 pod D에 `Tolerations` 추가

3. 특징

- Taint는 Node에, Toleratinos는 Pod에 세팅

4. Taints - Node

- `kubectl taint nodes [node-name] key=value:[taint-effect]` 명령을 통해 node에 taint 세팅
- `kubectl taint nodes node1 app=blue:NoSchedule`이 위의 예시의 taint
  
- taint-effect
  - 1. `NoSchedule` : Pod가 Node에 스케쥴링되지 않음
  - 2. `PreferNoSchedule` : 시스템은 Node에 Pod를 두는 것을 원하지 않지만 보장하지는 않음
  - 3. `NoExecute` : 실행불가. 새 Pod가 Node에 지정되지 않고 Node 위의 기존 Pod가 있다면 사용자가 거부하면 퇴거해야함

5. Tolerations - Pods

- Taints는 `kubectl taint nodes node1 app=blue:NoSchedule`
- Tolerations는 yaml 파일에 지정
  - tolerations의 값은 모두 더블 코드로 인코딩되어야함
  - 새로운 tolerations와 함께 pod가 생성되거나 업데이트될 때 효과 

`pod-definition.yml`
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
  #tolerations 추가
  tolerations:
    - key: app
      operator: Equal
      value: blue
      effect: NoSchedule
```

6. Taint - NoExecute

- worker node가 3개, pod가 4개로 가정
- Node1에 Taints 특히 taint-effect를 NoExecute, Pod D에 Tolerations를 진행한 경우, 기존에 존재했던 pod C는 퇴거 =Pod C kill
- 다른 Pod가 들어오지 않도록 하는 목적임을 인지할 것. Node1에 Pod D가 들어갈 수 있는 것이지 반드시 Pod D가 Node1에 할당되어야 하는 것은 아님 
- `Node Affinity`: 특정 Node에 하나의 Pod를 제한하는 것을 원하는 경우 사용
- 현재까지 worker node에서만 진행되었는데 클러스터엔 master node도 존재. 하지만 worker node에만 pod 할당 => Kubernetes 클러스터가 처음 설정되면 `master node에 taint가 자동으로 설정`되어 master node에 pod가 지정되는 것을 막음 
  - `kubectl describe node kubemaster | grep Taint`로 확인 가능

## Node Selectors

1. 예시: 3개의 Node Cluster 존재. 그 중 2개(Node-2,Node-3)는 작은 Node로 리소스가 적고, 1개(Node-1)는 큰 노드로 리소스가 많음. 클러스터에서 실행되는 다른 종류의 작업 존재

- 더 큰 Node에 더 높은 파워를 요구하는 데이터 프로세싱 작업 고정해야함. 작업이 추가 리소스를 요구할 경우 리소스를 줄 수 있는 유일한 노드이기에
- 현재는 pod가 어떤 node로든 갈 수 있음

2. 특정 Node에서만 작동하도록 Pod의 한계 설정 => `Node Selectors`, `Node Affinity`

- Node Selectors
  - Node Selector를 사용하기 위해 Pod 생성 전에 Node에 label을 먼저 붙여야함: `kubectl label nodes [NODE NAME] [LABEL KEY]=[LABEL VALUE]`
  - `kubectl label nodes nodes node-1 size=Large`
  - pod-definition file에 `nodeSelector` 추가. "size: Large" key-value 쌍은 Node에 할당된 label
  - scheduler는 이 label을 이용해 pod를 올릴 Node 찾아냄
  - Node Selector를 통해 Pod를 원하는 Node에 올렸지만, 훨씬 더 복잡한 요구 사항의 경우 한계 발생

`pod-definition.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  nodeSelector:
    size: Large
```

- Node Affinity
  - Node Selector보다 더 복잡한 요구 사항 적용 가능

## Node Affinity

1. Node Affinity (Node 선호도)

- Pod가 특정 Node에 Hosting되도록 하는 것이 목표로, 특정 Node에 Pod 배치를 제한하는 고급 기능 제공 

- 해당 Pod가 하나의 Node에 놓이게 됨. 해당 label 크기를 가진 지정된 값이 있는 Node에 놓이게 됨

- operator: `In`, `NotIn`, `Exists`

`pod-definition.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: In
                values:
                  - Large
```

2. Node Affinity Types
- Available: `requiredDuringSchedulingIgnoredDuringExecution`(Type1), `preferredDuringSchedulingIgnoredDuringExecution`(Type2)
  - DuringScheduling: Pod가 존재하지 않다가 처음으로 만들어지는 상태. Pod가 처음 생성되면 지정된 affinity(선호도) 규칙을 따라 pod를 올바른 node에 배치
    - 일치하는 label이 있는 Node를 사용할 수 없는 경우 => Type1은 주어진 선호도 규칙에 따라 해당 Pod를 Node에 배치. `해당 Node를 못 찾으면 Scheduling도 못 함`
    - Type2는 pod 배치가 작업 자체 실행보다 덜 중요한 경우. `일치하는 Node를 발견하지 못하면 Scheduler는 Node Affinity를 무시하고 해당 Pod를 아무 Node에 배치`
  - DuringExecution: Pod가 실행되고 환경이 변하면서 Affinity에 영향을 주는 상태. Ex) Node의 label 변경
    - Node Affinity의 변화는 스케쥴이 정해지면 영향을 미치지 않음
    - Type3는 Affinity 규칙에 부합하지 않는 Node에서 실행 중인 Pod는 쫓아냄

||DuringScheduling|DuringExecution|
|:--:|:--:|:--:|
|Type1|Required|Ignored|
|Type2|Preferred|Ignored|
|Type3|Required|Required|

- Planned: `requiredDuringSchedulingRequiredDuringExecution`(Type3)

## Taints & Tolerations vs Node Affinity

- Node의 label이 Blue, Red, Green, Other(2)이고, Pod도 Blue, Red, Green이 1개씩, Other이 2개 존재하는 경우

1. Taints and Tolerations

- Node에 Taints 설정, Pod에 Tolerations 설정
- Node Selector에 의해 Pod를 할당하는 경우 `원하는 Node가 아닌 곳에 Pod가 존재할 수 있음` => Node Affinity 사용

2. Taints/Tolerations and Node Affinity

- Taints/Tolerations를 이용해 다른 Pod가 우리 Node에 배치되는 것을 막음
- Node Affinity를 이용해 우리 Pod가 다른 Node에 배치되는 것을 막음

=> Pod를 원하는 Node에 고정시킬 수 있음

## Certification Tips - Student Tips

<https://www.linkedin.com/pulse/my-ckad-exam-experience-atharva-chauthaiwale/>

<https://medium.com/@harioverhere/ckad-certified-kubernetes-application-developer-my-journey-3afb0901014>

<https://github.com/lucassha/CKAD-resources>

## Labs 실습

1. Docker Image

Q?. image를 run할 때에는 Dockerfile에 정의된 ENTRYPOINT에 맞게 접근해야함
Q8

- `cd` 명령어를 통해 디렉토리 이동 & `pwd` 명령어로 현재 위치 확인
- Docker file을 통해 `webapp-color` 이름을 가지는 docker image 생성 => `docker build -t webapp-color .` : `.`은 코드 전체를 의미
  Q9. image를 실행하면서 container port 8080 port와 host port 8282 매핑
- `docker run -p 8282:8080 webapp-color`
- CTRL + C 로 중지 가능
  Q11. image의 base OS 확인
- `docker run python:3.6 cat /etc/*release*`
  Q14.
- base OS로 `python:3.6` 대신 `python:3.6-alpine` 사용
- alpine이란 가장 작은 사이즈로 정말 필요한 것들만 담겨져 있는 image

2. Commands and Arguments

Q2. What is the command used to run the pod ubuntu-sleeper?

- `kubectl get pod [POD NAME] -o yaml > [YAML NAME].yaml`로 pod에 따른 YAML 파일 생성 후 vi 편집기로 확인
  ![Alt text](image-2.png)

- Solution: `k describe [POD NAME]`으로 확인 가능

Q3. Create a pod with the ubuntu image to run a container to sleep for 5000 seconds. Modify the file ubuntu-sleeper-2.yaml

- command 하는 법
  - 1. []로 묶기. 이떄 command와 argument는 분리되어야함. 많이 사용하지 않는듯
       `command: ["sleep","5000"]`
  - 2. command에서 -로 분리. 3의 간소화 형식
  ```
  command:
    - "sleep"
    - "5000"
  ```
  - 3. command와 argument 각각 정의. 기본
  ```
  command: ["sleep"]
  args: ["5000"]
  ```

Q4. Create a pod using the file named ubuntu-sleeper-3.yaml. There is something wrong with it. Try to fix it!

- 아래와 같이 command의 child로 한 번에 정의 가능

![Alt text](image-3.png)

- Solution: `kubectl create` 명령어를 통해 생성해보고 오류메세지를 통해 유추

Q5. Update pod ubuntu-sleeper-3 to sleep for 2000 seconds.

- Solution: `kubectl edit pod [POD NAME]`으로 pod 수정 > 수정할 수 없다는 오류가 발생하고 "/tmp/kubectl-edit-2693604347.yaml"에 수정본 copy > `kubectl replace --force -f /tmp/kubectl-edit-2693604347.yaml`로 기존 pod 삭제 후 해당 파일로 새로운 pod 생성

Q8. Inspect the two files under directory webapp-color-2. What command is run at container startup? Assume the image was created from the Dockerfile in this directory.

- container 시작 시 yaml 파일에 지정된 command가 dockerfile을 무효화시키므로, yaml 파일을 확인해야함
- yaml 파일에 command의 parameter가 모두 재정의되었으므로 해당 command를 명령어로 실행

- Solution: `cat webapp-color-2/Dockerfile2`와 `cat webapp-color-2/webapp-color-pod.yaml`로 dockerfile과 yaml file 비교. yaml 파일에 재정의된 명령만 command로 실행되므로 `--color green`

Q9. What command is run at container startup?

- YAML파일에서 존재하는 모든 command와 argument 실행해야함

![Alt text](image-5.png)
![Alt text](image-4.png)

Q10. Create a pod with the given specifications. By default it displays a blue backgroud. Set the given command line arguments to change it to green.

- Solution: `kubectl run nginx --image=nginx -- <arg1> <arg2> ... <argN>` 사용해 argument 지정 가능 => `kubectl run webapp-green --image=kodekloud/webapp-color -- --color green`

3. ConfigMaps

- configmap을 cm으로 축약 가능

Q9. Create a new ConfigMap for the webapp-color POD. Use the spec given below.
ConfigName Name: webapp-config-map
Data: APP_COLOR=darkblue

`kubectl create cm webapp-config-map --from-literal=APP_COLOR=darkblue`

Q10. Update the environment variable on the POD to use the newly created ConfigMap. Note: Delete and recreate the POD. Only make the necessary changes. Do not modify the name of the Pod.

`pod.yaml`

```
spec:
  containers:
    - env:
        - name: APP_COLOR
          valueFrom:  #원래는 value였는데 해당 값을 configmap에서 가져옴
            configMapKeyRef:  #configMap 참조
              name: webapp-config-map #configmap name
              key: APP_COLOR  #참조할 key. 해당 key의 value를 configmap에서 가져옴
```

4. Secrets

Q6. The reason the application is failed is because we have not created the secrets yet. Create a new secret named db-secret with the data given below. You may follow any one of the methods discussed in lecture to create the secret.

CheckCompleteIncomplete

- `k create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123`

- `generic` 잊지 말 것

Q7. Configure webapp-pod to load environment variables from the newly created secret.

- `envFrom`은 모든 secret data 를 container environment variable로 정의. secret의 key가 pod의 environment variable name이 됨

```
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    name: webapp-pod
spec:
  containers:
    - name: webapp
      image: kodekloud/simple-webapp-mysql
      envFrom:
        - secretRef:
            name: db-secret
```

- Solution: `kubectl edit pod webapp-pod`에 envFrom을 추가한 후 `kubectl replace --force -f [복사본 경로]`

5. Security Contexts

Q1. What is the user used to execute the sleep process within the ubuntu-sleeper pod?

- `ps aux` 실행

- Solution: `kubectl exec [POD NAME] -- whoami`

Q2. Edit the pod ubuntu-sleeper to run the sleep process with user ID 1010.

- Solution: `kubectl delete pod ubuntu-sleeper --force` 하면 더 빠르게 

Q3. Update pod ubuntu-sleeper to run as Root user and with the SYS_TIME capability.

- Solution: spec>contianers 에 `securityContext: capabilities: add: ["SYS_TIME"]`

6. ServiceAccount

Q11. 

- Solution: serviceaccount에 대한 사용 권한 추가. `/var/rbac`에서 확인 가능: `cd /var/rbac` 후 `ls`


Q12. 

- Solution: /var/rbac 경로에서 `kubectl create token dashboard-sa`로 토큰 생성. 요즘은 토큰이 자동으로 생성되지 않아 별도로 생성해야함

Q13. deployment

- deployment는 spec>template>spec에 serviceAccountName 추가
- 이후 `kubectl apply -f [FILE명]`
  - pod와 달리 삭제할 필요 없이 실행만 시키면 됨

7. Resource Requirements

.

8. Taints and Toleratinos

Q4. Create a new pod with the nginx image and pod name as mosquito.

- `k run mosquito --image nginx`

Q7. Create another pod named bee with the nginx image, which has a toleration set to the taint mortein.

- Solution

```
...
spec:
  ...
  tolerations:
    - key: spray
      value: mortein
      effect: NoSchedule
      operator: Equal
```

Q8. Notice the bee pod was scheduled on node node01 despite the taint

- Solution: `kubectl get pods -o wide`로 node 확인

Q10. Remove the taint on controlplane, which currently has the taint effect of NoSchedule.

- Solution: `kubectl describe node`를 이용해 해당 Taints 전체 복사 > `kubectl taint node controlplane [복사본]-`. 마지막에 `-` 넣으면 taint 삭제 됨

9. Node Affinity

Q3. Apply a label color=blue to node node01

- Solution: `kubectl label node [NODE NAME] [KEY]=[VALUE]`으로 node에 label 생성


Q5. Which nodes can the pods for the blue deployment be placed on?

- Solution: `kubectl describe node [NODE NAME] | grep Taints`를 통해 각 Node의 Taints 확인. 두 Node 모두 Taints가 없으므로 blue deployment가 배치될 수 있음

Q6. Set Node Affinity to the deployment to place the pods on node01 only.

- Solution: spec의 child로 아래 코드 추가. Pod의 경우 parent spec을 의미하고, deployment의 경우 spec의 spec에 작성해야함

```
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: color
                operator: In
                values:
                  - blue
```