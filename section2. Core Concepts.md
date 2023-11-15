## Kubernetes Architecture

1. Nodes (= Minions)

- 쿠버네티스가 설치되는 물리적 가상머신

- Node는 작업자 머신으로 쿠버네티스가 컨테이너를 런칭하는 곳

- Node가 실패한다면 application이 다운되니 하나 이상의 Node 필요 => `Cluster` 등장

2. Cluster

- Cluster란 Node의 집합

- 하나의 Node가 실패해도 다른 Node에서 application에 접근 가능

- 다수의 Node는 Load 공유에도 도움

3. Master Node

- 쿠버네티스가 설치된 또 다른 Node

- Cluster에서 Node 관리 
    - 실제 orchestration 책임 
    - Cluster 멤버에 대한 정보 
    - Monitoring 관리 
    - Node가 실패하면 실패한 Node의 작업을 다른 작업자 Node로 옮기는 역할

4. Components 
: System에 Kubernetes를 설치할 구성 요소

- `API Server`
    - Kubernetes의 프론트엔드처럼 작동
    - user, management device, command line interface 모두 API Server에 통신해 Kubernetes Cluster와 상호작용

- `etcd`
    - 분산되고 신뢰할 수 있는 Key-value 저장소
    - Kubernetes가 Cluster 관리에 사용되는 모든 데이터를 저장하는데 사용
    - Ex. Cluster에 Node와 Master가 여러 개씩 존재하는 경우, Cluster 내 모든 Node에 관한 정보를 분산 방식으로 저장  
    - Cluster 내에서 잠금장치 구현 => Master 간의 충돌이 없도록

- `kubelet`
    - Cluster의 각 Node에서 실행되는 agent
    - 예상대로 Container가 Node에서 실행되는지 확인

- `Container Runtime`
    - Container를 실행하는 데 사용되는 기본 소프트웨어
    - Ex. Docker

- `Controller`
    - orchestration을 지휘하는 두뇌 
    - Node, Container, Endpoint가 다운될 때 인지하고 대응하는 역할로, 해당 경우 새 container를 들여올지 결정

- `Scheduler`
    - 다중 노드에 걸친 작업이나 Container 배포 담당
    - 새로 생성된 Container를 찾아 Node에 할당 

5. Master vs Worker Nodes
: Server의 유형은 Master와 Worker로 나뉨

- Master Node
    - `kube-apiserver`를 가지기에 Master Node가 됨
    - 수집된 모든 Node의 정보는 `etcd`에 Key-value 형태로 저장
    - `controller`와 `scheduler` 존재 

- Worker Node
    - 모든 worker node에는 `kubelet`이 있어 Master Node와 상호 작용 => Worker Node의 health 정보를 제공하고 Master Node가 요청한 작업 수행

    - Container(ex.Docker container)가 Hosting 되는 곳
    - System에서 Docker container를 실행하려면 Container Runtime이 설치되어야 함
    - 본 강의에서는 Container Runtime으로 Docker 사용할 것

6. kubectl

- Cluster 상의 application을 배포 및 관리하는데 사용
- Cluster 정보를 얻고 Cluster 내의 다른 Node의 상태를 얻을 수 있고 많은 것을 관리할 수 있음
- `kubectl run` : cluster에 application을 배포
- `kubectl cluster-info` : cluster 정보 조회
- `kubectl get nodes` : cluster 내의 모든 node를 나열

## Docker vs ContainerD

1. Docker와 Kubernetes

- container 초기에는 Docker가 가장 지배적인 container 도구가 됨 
- 이후 Kubernetes가 Docker와의 orchestration`만`을 담당
- Kubernetes가 container orchestrator로 인기가 높아졌고 다른 container runtime이 들어오길 원함 => Kubernetes가 `Container Runtime Interface(CRI)` 도입. CRI를 사용해 `OCI 표준을 준수`하는 어떤 공급업체든 Kubernetes의 container runtime을 지원해 Kubernetes의 container runtime으로 작업하게 해줌
    - Ex. ctr, nerdctl, crictl

- Docker는 CRI 등장 전부터 사용했기에 표준을 지원하려고 만든 것이 아님
- Kubernetes가 `dockershim` 도입 => container runtime interface 밖에서 Docker를 지원하는 임시 방편
- Docker에는 Container runtime만 가지는 것이 아님 
    - Docker CLI, Docker API, Build(image 구축을 돕는 빌드 도구), Volumes, Auth, Security, ContainerD(container runtime. daemon)
    - `Contianerd`는 CRI 호환이 가능하고 다른 runtime처럼 Kubernetes와 직접적으로 작업 가능
    - Containerd는 Docker와 별도로 자체 Runtime으로 사용 가능
- dockershim을 유지하기 불편 => `Kubernetes가 dockershim 지원 중단`
- docker image는 OCI의 imagespec을 따르기에 계속 작동. container로 계속 작업하지만 Docker 자체는 Kubernetes 지원 runtime에서 제거  

2. OCI (Open Container Initiative)

- `imagespec`과 `runtimespec`으로 구성

- imagespec은 image를 어떻게 만들어야 하는지 사양에 대한 정의

- runtimespec은 container runtime 개발에 대한 표준 정의

3. Containerd

- Docker의 일부지만 현재는 독립된 프로젝트
- Docker의 다른 기능이 필요없는 경우 전체를 설치할 필요없이 containerd만을 설치해 container만 설치 가능 
- Docker가 설치되어있지않고 containerd만 설치된 경우(container만 장착된 container) 실행 방법 
    - `ctr` 명령어 사용
    - `nerdctl` 명령어 사용
        - ctr 명령어에 비해 좋은 대안    
        - docker와 매우 유사한 명령어로 container를 위한 Docker 명령어와 비슷한 CLI. `docker` 대신 `nerdctl`로 변경해 사용

4. CLI 명령어

- `ctr` 명령어 사용
    - ctr 명령어는 `디버깅 container를 위해서만` 만들어졌고 제한된 기능만 제공하기에 사용자 친화적이지 않음
    - ctr 명령어는 image를 가져오는 것과 같은 container 관련 작업을 수행할 때 사용
    - `ctr images pull [image주소]` : image pull
    - `ctr run [image주소]` : container 실행
- `nerdctl` 명령어 사용
    - ctr 명령어에 비해 좋은 대안    
    - docker와 매우 유사한 명령어로 container를 위한 Docker 명령어와 비슷한 CLI. `docker` 대신 `nerdctl`로 변경해 사용
    - docker가 지원하는 대부분의 옵션을 지원하고 이외의 container에 구현된 최신 기능에 대한 접근 권한 제공
    - `nerdctl run` : container 생성
    - `nerdctl run --name webserver -p 80:80 -d nginx` : p 옵션으로 포트 매핑
- `crictl`
7:35