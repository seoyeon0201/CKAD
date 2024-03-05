## Q1
Create a Persistent Volume called log-volume. It should make use of a storage class name manual. It should use RWX as the access mode and have a size of 1Gi. The volume should use the hostPath /opt/volume/nginx

Next, create a PVC called log-claim requesting a minimum of 200Mi of storage. This PVC should bind to log-volume.

Mount this in a pod called logger at the location /var/www/nginx. This pod should use the image nginx:alpine.

## Q2
We have deployed a new pod called secure-pod and a service called secure-service. Incoming or Outgoing connections to this pod are not working.
Troubleshoot why this is happening.

Make sure that incoming connection from the pod webapp-color are successful.


Important: Don't delete any current objects deployed.

## Q3
Create a pod called time-check in the dvl1987 namespace. This pod should run a container called time-check that uses the busybox image.

Create a config map called time-config with the data TIME_FREQ=10 in the same namespace.
The time-check container should run the command: while true; do date; sleep $TIME_FREQ;done and write the result to the location /opt/time/time-check.log.
The path /opt/time on the pod should mount a volume that lasts the lifetime of this pod.

## Q4
Create a new deployment called nginx-deploy, with one single container called nginx, image nginx:1.16 and 4 replicas.
The deployment should use RollingUpdate strategy with maxSurge=1, and maxUnavailable=2.

Next upgrade the deployment to version 1.17.

Finally, once all pods are updated, undo the update and go back to the previous version.

## Q5
Create a redis deployment with the following parameters:

Name of the deployment should be redis using the redis:alpine image. It should have exactly 1 replica.

The container should request for .2 CPU. It should use the label app=redis.

It should mount exactly 2 volumes.


a. An Empty directory volume called data at path /redis-master-data.

b. A configmap volume called redis-config at path /redis-master.

c. The container should expose the port 6379.


The configmap has already been created.