## Q1
Create a Persistent Volume called log-volume. It should make use of a storage class name manual. It should use RWX as the access mode and have a size of 1Gi. The volume should use the hostPath /opt/volume/nginx

Next, create a PVC called log-claim requesting a minimum of 200Mi of storage. This PVC should bind to log-volume.

Mount this in a pod called logger at the location /var/www/nginx. This pod should use the image nginx:alpine.

## A1

`log-volume.yaml`
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-volume
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/opt/volume/nginx"
```

`log-claim.yaml`
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-claim
spec:
  storageClassName: manual # Empty string must be explicitly set otherwise default StorageClass will be set
  volumeName: log-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
```

- spec.accessModes는 앞에서 주어진 값과 일치

`logger.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: logger
spec:
  containers:
    - name: myfrontend
      image: alpine
      volumeMounts:
      - mountPath: "/var/www/nginx"
        name: log-claim
  volumes:
    - name: log-claim
      persistentVolumeClaim:
        claimName: log-claim
```

- spec.containers.`volumeMounts.name`이 spec.`volumes.name`과 일치해야함

#### 헷갈리는 부분 CHECK
Q. Pod의 spec.containers.volumeMounts.mountPath와 PV의 spec.hostPath.path 차이

A. 
- container는 기본적으로 stateless한 상태이기에 자유롭게 다른 노드로 옮길 수 있지만 동시에 컨테이너가 삭제되면 저장된 데이터가 사라진다는 단점 존재 => container에 문제가 발생하더라도 데이터 보존을 위한 저장소 Volume 필요 

- Pod가 실행될 때의 데이터를 저장하는 volume으로, emptyDir, hostPath, local, NFS 존재

- PV는 볼륨 자체를 의미하며 클러스터 내에서 자원으로 다루고 지정한 위치에 데이터 존재

- 즉, PV는 외부 데이터 저장소를 의미하고 Pod에 mountPath는 pod가 실행되면서 변경하는 데이터를 저장하는 저장소 

Q. PV와 PVC

A. PV라는 타입의 volume을 생성하고 PVC로 해당 PV를 가리키며, 다른 object는 PVC를 통해 PV 사용

## Q2
We have deployed a new pod called secure-pod and a service called secure-service. Incoming or Outgoing connections to this pod are not working.
Troubleshoot why this is happening.

Make sure that incoming connection from the pod webapp-color are successful.


Important: Don't delete any current objects deployed.

## A2.

`k exec -it webapp-color -- sh`

`nc -v -z -w 2 secure-service 80`
- nc는 netcat 명령어를 의미하며 TCP 또는 UDP 통신 테스트할 떄 쓰임
    - `-v` : 자세한 정보 출력 
    - `-z` : 성공 여부만 출력하고 데이터는 전송하지 않도록 함 
    - `-w` : 타임 아웃 시간을 초 단위로 지정하도록 하여 해당 시간 이후에 타임 아웃되도록 함
    - 80 포트는 `k get svc`를 통해 조회 가능
- Operation timed out을 통해 연결에 문제가 있음을 알 수 있음 => service를 통해 pod 접근 불가능

`k get netpol`
`k describe netpol`
- 네트워크 정책 확인 => `Allowing ingress traffic`에 아무것도 없어 문제 발생

`k get pod --show-labels`
- pod의 label 조회

`k get netpol default-deny -o yaml > netpol.yaml`

`netpol.yaml`
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  creationTimestamp: "2024-03-06T17:05:09Z"
  generation: 1
  name: default-agree
  namespace: default
  resourceVersion: "2709"
  uid: abbbb54b-204a-4ae0-86d3-8876f22f72de
spec:
  podSelector:
    matchLabels:
      run: secure-pod
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: webapp-color
    ports:
    - protocol: TCP
      port: 80
```

- spec.podSelector는 pod의 labels

## Q3
Create a pod called time-check in the dvl1987 namespace. This pod should run a container called time-check that uses the busybox image.

Create a config map called time-config with the data TIME_FREQ=10 in the same namespace.
The time-check container should run the command: while true; do date; sleep $TIME_FREQ;done and write the result to the location /opt/time/time-check.log.
The path /opt/time on the pod should mount a volume that lasts the lifetime of this pod.

## A3

`k create namespace dvl1987`

`time-config.yaml`
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-config
  namespace: dvl1987
data:
  TIME_FREQ: "10"
```

`time-check`
```
apiVersion: v1
kind: Pod
metadata:
  name: time-check
  namespace: dvl1987
spec:
  containers:
  - name: time-check
    image: busybox
    env:
      - name: TIME_FREQ
        valueFrom:
          configMapKeyRef:
            name: time-config
            key: TIME_FREQ
    command: ['sh', '-c', 'while true; do date; sleep $TIME_FREQ;done > /opt/time/time-check.log']
    volumeMounts:
      - name: time-vol
        mountPath: /opt/time
  volumes:
  - name: time-vol
    emptyDir: {}
```

- configMap의 데이터를 사용하기 위해 env로 key를 받고 valueFrom으로 실제 configMap에서 값 가져옴

- volume은 `기본 volume인 emptyDir` 사용

#### 헷갈리는 부분 CHECK

Q. ConfigMap 

A. 
- 데이터를 key-value 쌍으로 저장하는 데 사용 
- value 값은 반드시 `문자열로 ""로 묶어 사용` 
- Pod에서 데이터를 사용할 때에는 `env`로 데이터 받아와 사용


## Q4
Create a new deployment called nginx-deploy, with one single container called nginx, image nginx:1.16 and 4 replicas.
The deployment should use RollingUpdate strategy with maxSurge=1, and maxUnavailable=2.

Next upgrade the deployment to version 1.17.

Finally, once all pods are updated, undo the update and go back to the previous version.

## A4.

`nginx-deploy.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: nginx-deploy
 labels:
   app: nginx
spec:
 replicas: 4
 selector:
   matchLabels:
     app: nginx
 template:
   metadata:
     labels:
       app: nginx
   spec:
     containers:
     - name: nginx
       image: nginx:1.16
       ports:
       - containerPort: 80
 strategy:
   type: RollingUpdate
   rollingUpdate:
     maxSurge: 1
     maxUnavailable: 2
```

`k set image -f nginx-deploy.yaml nginx=nginx:1.17`

`kubectl rollout undo deployment`

## Q5
Create a redis deployment with the following parameters:

Name of the deployment should be redis using the redis:alpine image. It should have exactly 1 replica.

The container should request for .2 CPU. It should use the label app=redis.

It should mount exactly 2 volumes.


a. An Empty directory volume called data at path /redis-master-data.

b. A configmap volume called redis-config at path /redis-master.

c. The container should expose the port 6379.


The configmap has already been created.

## A5.

`redis.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis-pod
        image: redis:alpine
        resources:
          requests:
            cpu: "0.2"
        ports:
        - containerPort: 6379
        volumeMounts:
        - mountPath: /redis-master-data
          name: data
        - mountPath: /redis-master
          name: redis-config
      volumes:
      - name: data
        emptyDir: {}
      - name: redis-config
        configMap:
          name: redis-config
```