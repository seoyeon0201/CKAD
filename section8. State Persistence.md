## Volumes

1. Volumes

- Docker Volume

  - 일시적으로 단기간에만 사용 가능
  - 컨테이너 내의 데이터 처리가 필요할 때 호출되고 끝나면 폐기 (컨테이너 내의 데이터도 폐기)

- Container Volume

  - container로 데이터를 처리하기 위해 container 생성시 volume 생성
  - container로 처리한 데이터를 volume에 저장 후 영구적으로 보존
    - container가 삭제되어도 생성된 데이터나 처리된 데이터는 남아있음

- Kubernetes Volume
  - pod에 volume 부착되어 pod 삭제해도 데이터 보존
  - pod에서 생성된 데이터가 volume에 저장

2. Volumes & Mounts

- `/opt/number.out` 파일에 0부터 100까지의 랜덤한 수 작성
- volume에 생성된 파일을 host(본 node)에 있는 `/data`이름의 directory에 저장
- container에서 접근할 volume이 생성되면 container 내부의 디렉토리에 해당 volume mount
  - container 내의 /opt 경로가 data-volume과 mount되어 host의 /data 경로에 본 container에서 처리한 데이터가 존재
- pod가 삭제되어도 host에 데이터 남아있음

```
apiVersion: v1
kind: Pod
metadata:
    name: random-number-generator
spec:
    containers:
        - image: alpine
          name: alpine
          command: ["/bin/sh","-c"]
          args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
          volumeMounts:
            - mountPath: /opt
              name: data-volume

    volumes:
        - name: data-volume
          hostPath:
            path: /data
            type: Directory
```

3. Volume Types

- 다중 노드의 경우 hostPath 사용 X

  - pod는 모든 node에서 /data 디렉토리를 사용하고 동일한 데이터이길 기대하지만 그렇지 않음
  - 다른 서버에 존재하기에 복제된 외부 cluster storage solution을 구성하지 않으면 같은 서버가 아님

- External Cluster Storage Solution
  - ex. Amazon Elastic, NFS, ClusterFS, Flocker, Ceph, Scaleio, Fiber Channel, Azure Disc File

`amazon elastic`

```
volumes:
    - name: data-volume
      awsElasticBlockStore:
        volumeID: [VOLUME ID]
        fsType: ext4
```

## Persistent Volumes

1. Previous

- 이전 세션에서 pod definition file에서 volume 정의
- 많은 pod를 배포하는 경우 pod마다 volume을 만들어야하기에 불편

- `저장소를 중앙에서 관리` => Persistent Volumes
  - 관리자가 거대한 storage를 생성한 후 사용자의 요구대로 일부를 나누어감

2. Persistent Volume

- Persistent Volume은 cluster에 존재하는 거대한 storage
- 사용자는 `Persistent Volume Claim`에 따라 PV에서 storage 선택 가능

3. Persistent Volume definition file

- `spec.accessModes` : host에 volume이 어떻게 mount되어야하는지 정의
  - ReadOnlyMany / ReadWriteOnce / ReadWriteMany
- `spec.capacity` : 1GB로 설정된 PV을 위해 예약되는 저장소의 양
- `spec.[VOLUME TYPE]`
  - VOLUME TYPE은 hostPath(로컬 디렉토리 노드의 저장소 사용) / awsElasticBlockStore

`pv-definition.yaml`

```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-vol1
spec:
    accessModes:
        - ReadWriteOnce
    capacity:
        storage: 1Gi
    #(1)hostPath
    hostPath:
        path: /tmp/data
    #(2)aws elastic
    awsElasticBlockStore:
        volumeID: [VOLUME ID]
        fsType: ext4
```

## Persistent Volume Claims

1. Persistent Volume vs Persistent Volume Claims

- Persistent Volume과 Persistent Volume Claim은 Kubernetes namespace에 존재하는 서로 다른 object
- 관리자 => Persistent Volume 생성
- 사용자 => Persistent Volume Claim 생성

2. Persistent Volume Claim 과정

- 과정: PVC 생성 > PV와 PVC binding 
- 이때 모든 PVC는 하나의 volume으로 묶여있음
- Binding process동안 Kubernetes는 PV를 찾음
    - `Sufficient Capacity`, `Access Modes`, `Volume Modes`, `Storage Class`에 따라 탐색

- 다양한 case
    - 단일 PVC에 매치 가능한 PV가 여러 개인데 그 중 특정 volume을 원하는 경우, Label과 Selector를 이용해 지정
        - PV definition: `labels: name: my-pvc`
        - PVC definition: `selector: matchLabels: name: my-pvc`
    - 다른 모든 기준이 일치하고 더 나은 선택지가 없는 경우, 작은 PVC는 더 큰 용량의 PV와 binding 될 수 있음

- PV와 PVC는 1:1 관계이므로 PV가 남은 경우 다른 PVC가 해당 PV를 사용할 수 없음
- 사용가능한 PV가 없는 경우 PVC는 pending 상태이고, 사용가능한 PV가 생기면 자동으로 binding

3. Persistent Volume Claim definition

`pvc-definition.yaml`
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: myclaim
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 500Mi
```

- 해당 PVC는 `pv-definition.yaml`을 이용해 생성한 pv와 binding

4. Delete PVCs
- PVC가 삭제된 경우 PV는 기본으로 Retain 상태(유지)이기에 다른 PVC에 의해 재사용될 수 없음
    - `persistentVolumeReclaimPolicy: Retain`
    - `persistentVolumeReclaimPolicy: Delete`로 설정한 경우 PVC가 삭제되면 PV에 생성된 데이터 자동으로 모두 삭제
    - `persistentVolumeReclaimPolicy: Retain`로 설정한 경우 PV의 데이터가 다른 PVC에 사용되기 전에 삭제 


## Using PVCs in Pods

1. PVC 사용하는 경우 pod definition

- `spec.volumes.persistentVolumeClaim` 설정

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim

```

## Storage Classes

**Storage Class를 포함하여 이번 섹션의 남은 주제들은 CKAD에는 나오지 않음**

1. Static Provisioning Volume

- Google Cloud Persistence Disk에서 PVC 생성

- 이 때 PV가 생성되기 전 **Google Cloud에 disk를 생성해야함**
`gcloud beta compute disks create \ --size 1GB --region us-east1 pd-disk`
    - application이 저장소를 필요로 할 때마다 먼저 수동으로 google cloud disk provision해야함
- 그 후 수동으로 PVC definition file 생성
    - 방금 생성한 disk와 같은 이름 사용

2. Dynamic Provisioning

- Storage Class로 Google storage 같은 provisioner 정의 가능
- Google Cloud에서 자동으로 storage provision하고 pod에 연결 => `Dynamic Provisioning`

`sc-definition.yaml`
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: google-storage
provisioner: kubernetes.io/gce-pd
```

- SC를 사용하면 더이상 PV definition이 필요하지 않음 => sc를 생성하면 pv와 관련된 저장소가 생성되기 때문
    - sc를 생성하면 자동으로 pv가 생성되기에 definition할 필요가 없는 것
    - 아예 pv가 없는 것이 아님
- sc를 사용하면 위에서 말한 수동으로 disk 생성하는 과정을 거치지 않아도 됨
- Storage Class 사용
    - `pvc-definition.yaml`에 `spec.storageClassName`으로 `sc-definition.yaml`의 sc 이름을 작성하면 sc 사용 가능
    - pvc가 생성되면 그와 관련된 sc는 provisioner를 사용해 GCP에서 요구하는 크기로 새 disk provisioning 
    - pv를 생성해 pvc와 결합

3. Storage Class

- GCP volume을 사용하기 위해 GCE provisioner 사용
- provisioner에 따라 parameter를 지정할 수 있음

`sc-definition.yaml`
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: google-storage
provisioner: kubernetes.io/gce-pd
parameters:
    type: pd-standard
    replication-type: none
```

## Why Stateful Sets?

1. 상황

- 물리적 서버에 데이터베이스 서버 배포
- 고가용성을 위해 기존 데이터베이스의 데이터를 새 서버의 새 데이터베이스로 복제

2. 복제 과정
- 1. Setup master first and then slaves
- 2. Clone data from the master to slave-1
- 3. Enable continuous replication from master to slave-1
- 4. Wait for slave-1 to be ready
- 5. Clone data from slave-1 to slave-2
- 6. Enable continuous replication from master to slave-2
- 7. Configure Master Address on Slave

3. Deployment 사용

- 복제에서는 master와 slave 순서를 고려해 진행되지만, Deployment는 동시에 이루어지므로 순서를 보장할 수 없음
- master와 slave를 나눠야해 master에는 변하지 않는 hostname이나 IP address가 주어져야하는데, deployment의 pod IP address는 dynamic하게 할당되므로 의존할 수 없음. pod를 다시 만들 때 IP address가 바뀔 수 있음. pod name은 무작위로 지어지기 때문에 안 됨 => pod 중 하나를 master로 설정할 수 없으며 (7)에서 slave가 master IP를 가리켜야하는데 가리킬 수 없음  

4. StatefulSet 사용

- Pod가 순차적으로 생성되어 pod의 순서에 따라 master, slave 나눌 수 있음
- 각 pod는 고유한 index를 가지게 되어 고유한 이름을 가지게 되어 위 단계를 진행할 수 있음
- pod가 삭제되고 다시 생성되어도 동일한 pod name을 가져 (7)에서 해당 pod를 가리킬 수 있음

## Stateful Sets Introduction

1. StatefulSet

- instance가 특정 순서로 나타나야하는 경우 & 안정적인 이름을 가져야하는 경우 사용

- definition file에 `spec.serviceName` 존재

`statefulset-definition.yml`
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
    name: mysql
    labels:
        app: mysql
spec:
    template:
        metadata:
            labels:
                app: mysql
        spec:
            containers:
                - name: mysql
                  image: mysql
    replicas: 3
    selector:
        matchLabels:
            app: mysql
    serviceName: mysql-h
    podManagementPolicy: Parallel
```

2. Statefulset 특징

-  각 pod는 network에서 안정적이고 고유한 DNS 레코드, 네트워크 ID를 얻음 
    - 다른 application이 pod에 접근할 때 사용할 수 있음
- scale up, down이 쉬움
    - `kubectl scale statefulset mysql --replicas=5`로 5개로 scale up
    - 이 경우 새 instance를 이전 instance에 clone할 수 있음
    - scale up시 pod가 순서대로 생겨나고 scale down 또는 삭제 시 뒤에서부터 삭제
- `podManagementPolicy: Parallel` : 순서대로 하지 않으려면 설정

## Headless Services

## Storage in StatefulSets

## Labs 실습

1. Persistent Volumes

Q2

- Sol: `kubectl exex webapp -- cat /log/app.log` => webapp이름의 pod 실행 후 /log/app.log 경로의 로그 확인

Q3

- Sol: volume이 `kube-api-access-lmntb`는 기본 volume으로 pod에 생성된 가장 기본 storage이르모 pod 삭제 시 삭제됨


Q4

- Sol: `kubectl edit pod webapp`에서 코드 수정> copy본 복사> `k replace --force -f [COPY.yaml]`로 재생성. `cd /var/log/webapp` > `ls` 에 존재하는 app.log가 host에 저장된 storage

Q16

- Sol: pod가 pvc를 사용하고 있는 경우 pvc는 삭제되지 않음 => Terminating 상태

**object 생성 시 Kubernetes 공식 페이지 참고해 definition file 작성 연습**