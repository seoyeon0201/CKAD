## Q1

We have deployed a few pods in this cluster in various namespaces. Inspect them and identify the pod which is not in a Ready state. Troubleshoot and fix the issue.

Next, add a check to restart the container on the same pod if the command ls /var/www/html/file_check fails. This check should start after a delay of 10 seconds and run every 60 seconds.


You may delete and recreate the object. Ignore the warnings from the probe.

## A1

## Q2

Create a cronjob called dice that runs every one minute. Use the Pod template located at /root/throw-a-dice. The image throw-dice randomly returns a value between 1 and 6. The result of 6 is considered success and all others are failure.

The job should be non-parallel and complete the task once. Use a backoffLimit of 25.

If the task is not completed within 20 seconds the job should fail and pods should be terminated.

## A2

`cronjob.yaml`
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dice
spec:
  schedule: "1 * * * *"
  jobTemplate:
    spec:
      activeDeadlineSeconds: 20
      backoffLimit: 25
      template:
        spec:
          containers:
          - name: dice
            image: throw-dice
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: Never
```

## Q3

Create a pod called my-busybox in the dev2406 namespace using the busybox image. The container should be called secret and should sleep for 3600 seconds.

The container should mount a read-only secret volume called secret-volume at the path /etc/secret-volume. The secret being mounted has already been created for you and is called dotfile-secret.

Make sure that the pod is scheduled on controlplane and no other node in the cluster.

## A3

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
  - name: my-busybox
    image: busybox
    ports:
    - containerPort: 80
    command: ['sh', '-c', sleep 3600']
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret
```

## Q4

Create a single ingress resource called ingress-vh-routing. The resource should route HTTP traffic to multiple hostnames as specified below:

The service video-service should be accessible on http://watch.ecom-store.com:30093/video

The service apparels-service should be accessible on http://apparels.ecom-store.com:30093/wear


Here 30093 is the port used by the Ingress Controller

## A4

## Q5

A pod called dev-pod-dind-878516 has been deployed in the default namespace. Inspect the logs for the container called log-x and redirect the warnings to /opt/dind-878516_logs.txt on the controlplane node

## A5