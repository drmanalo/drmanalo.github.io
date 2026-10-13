+++
date = "2026-10-12T16:57:26+01:00"
draft = false
title = "CKAD: Fourth Mock Challenge"
tags = ["linux","kubernetes","docker"]
+++

# Task

Build a highly available `redis-cluster` based on the given architecture diagram.

![redis-cluster](../challenge4.png)

<!--more-->

## Inspect redis-cluster-configmap

```
➜  kubectl describe configmap redis-cluster-configmap
Name:         redis-cluster-configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis.conf:
----
cluster-enabled yes
cluster-require-full-coverage no
cluster-node-timeout 15000
cluster-config-file /data/nodes.conf
cluster-migration-barrier 1
appendonly yes
protected-mode no
update-node.sh:
```

## Label node01 for affinity

```
➜  kubectl label node node01 app=redis-cluster
node/node01 labeled

➜  kubectl get nodes --show-labels 
NAME           STATUS   ROLES           AGE   VERSION   LABELS
controlplane   Ready    control-plane   81m   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controlplane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
node01         Ready    <none>          81m   v1.30.0   app=redis-cluster,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01,kubernetes.io/os=linux
```

## Persistent volume

```
➜  for name in redis{01..06}; do
> kubectl create -f - <<EOF
> apiVersion: v1
kind: PersistentVolume
metadata:
  name: $name
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /$name
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: app
          operator: In
          values:
          - redis-cluster
> EOF
> done
persistentvolume/redis01 created
persistentvolume/redis02 created
persistentvolume/redis03 created
persistentvolume/redis04 created
persistentvolume/redis05 created
persistentvolume/redis06 created
```

## redis-cluster.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-service
  labels:
    app: redis-cluster
spec:
  ports:
  - port: 6379
    name: client
    targetPort: 6379
  - port: 16379
    name: gossip
    targetPort: 16379
  selector:
    app: redis-cluster
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  selector:
    matchLabels:
      app: redis-cluster
  serviceName: redis-cluster-service
  replicas: 6
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 6369
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster-configmap
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

## Create cluster replica

```
➜  kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 {end}')
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.17.1.5:6379 to 172.17.1.2:6379
Adding replica 172.17.1.6:6379 to 172.17.1.3:6379
Adding replica 172.17.1.7:6379 to 172.17.1.4:6379
M: daa27882d3e5fd34bdc8a266a33ab104e5fc14d9 172.17.1.2:6379
   slots:[0-5460] (5461 slots) master
M: a79ada8ea5d64dc1aa07c866230dcf6b110cd877 172.17.1.3:6379
   slots:[5461-10922] (5462 slots) master
M: 702406341d135a9e07273b7d5fdaa113cdd8f34a 172.17.1.4:6379
   slots:[10923-16383] (5461 slots) master
S: 432fbdfd825137ba62d59ea73d9641198d31ff71 172.17.1.5:6379
   replicates daa27882d3e5fd34bdc8a266a33ab104e5fc14d9
S: 5afabde457cff45513247f40ba12bf5e25322fb2 172.17.1.6:6379
   replicates a79ada8ea5d64dc1aa07c866230dcf6b110cd877
S: dc9ec53a9c0beccc346c2d6c636e526c263990ef 172.17.1.7:6379
   replicates 702406341d135a9e07273b7d5fdaa113cdd8f34a
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
....
>>> Performing Cluster Check (using node 172.17.1.2:6379)
M: daa27882d3e5fd34bdc8a266a33ab104e5fc14d9 172.17.1.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: dc9ec53a9c0beccc346c2d6c636e526c263990ef 172.17.1.7:6379
   slots: (0 slots) slave
   replicates 702406341d135a9e07273b7d5fdaa113cdd8f34a
S: 5afabde457cff45513247f40ba12bf5e25322fb2 172.17.1.6:6379
   slots: (0 slots) slave
   replicates a79ada8ea5d64dc1aa07c866230dcf6b110cd877
M: a79ada8ea5d64dc1aa07c866230dcf6b110cd877 172.17.1.3:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 702406341d135a9e07273b7d5fdaa113cdd8f34a 172.17.1.4:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 432fbdfd825137ba62d59ea73d9641198d31ff71 172.17.1.5:6379
   slots: (0 slots) slave
   replicates daa27882d3e5fd34bdc8a266a33ab104e5fc14d9
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## Verifications

```
➜  kubectl get service
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
kubernetes              ClusterIP   172.20.0.1      <none>        443/TCP              94m
redis-cluster-service   ClusterIP   172.20.157.16   <none>        6379/TCP,16379/TCP   71s

➜  kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
redis-cluster-0   1/1     Running   0          74s
redis-cluster-1   1/1     Running   0          69s
redis-cluster-2   1/1     Running   0          66s
redis-cluster-3   1/1     Running   0          63s
redis-cluster-4   1/1     Running   0          60s
redis-cluster-5   1/1     Running   0          57s

➜  kubectl get pvc
NAME                   STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-redis-cluster-0   Bound    redis04   1Gi        RWO                           <unset>                 78s
data-redis-cluster-1   Bound    redis02   1Gi        RWO                           <unset>                 73s
data-redis-cluster-2   Bound    redis05   1Gi        RWO                           <unset>                 70s
data-redis-cluster-3   Bound    redis03   1Gi        RWO                           <unset>                 67s
data-redis-cluster-4   Bound    redis06   1Gi        RWO                           <unset>                 64s
data-redis-cluster-5   Bound    redis01   1Gi        RWO                           <unset>                 61s

➜  kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
redis01   1Gi        RWO            Retain           Bound    default/data-redis-cluster-5                  <unset>                          15m
redis02   1Gi        RWO            Retain           Bound    default/data-redis-cluster-1                  <unset>                          15m
redis03   1Gi        RWO            Retain           Bound    default/data-redis-cluster-3                  <unset>                          15m
redis04   1Gi        RWO            Retain           Bound    default/data-redis-cluster-0                  <unset>                          15m
redis05   1Gi        RWO            Retain           Bound    default/data-redis-cluster-2                  <unset>                          15m
redis06   1Gi        RWO            Retain           Bound    default/data-redis-cluster-4                  <unset>                          15m
```