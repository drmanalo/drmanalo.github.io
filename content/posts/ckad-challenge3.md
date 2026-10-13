+++
date = "2026-10-12T16:36:26+01:00"
draft = false
title = "CKAD: Third Mock Challenge"
tags = ["linux","kubernetes","docker"]
+++

## Task
Deploy the given architecture to `vote` namespace.

![vote-namespace](../challenge3.png)

<!--more-->

## Deployments

```
➜ kubectl create deployment vote --image dockersamples/examplevotingapp_vote --replicas 1 -n vote

➜ kubectl create deployment redis --image redis:alpine --replicas 1 --dry-run=client -o yaml > redis.yaml

➜ kubectl create deployment worker --image dockersamples/examplevotingapp_worker --replicas 1 -n vote

➜ kubectl create deployment db --image postgres:15-alpine --replicas 1 --dry-run=client -o yaml > db.yaml

➜ kubectl create deployment result --image dockersamples/examplevotingapp_result --replicas 1 -n vote
```

### redis.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis
  name: redis
  namespace: vote
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
      - image: redis:alpine
        name: redis
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        emptyDir: {}
```

### db.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: db
  name: db
  namespace: vote
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  strategy: {}
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: db-data
        emptyDir: {}
```

### Check deployments

```
➜  kubectl get deployments -n vote
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
db       1/1     1            1           30m
redis    1/1     1            1           27m
result   1/1     1            1           44m
vote     1/1     1            1           44m
worker   1/1     1            1           18m
```

## Service

```
➜  kubectl expose deployment vote --name vote --port 8080 --dry-run=client -n vote -o yaml > vote-service.yaml

➜  kubectl expose deployment redis -n vote --name redis --port 6379 --type ClusterIP

➜  kubectl expose deployment db -n vote --name db --port 5432 --type ClusterIP

➜  kubectl expose deployment result --name result --port 8081 --type NodePort --dry-run=client -o yaml > result-service.yaml -n vote
```

### vote-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: vote
  name: vote
  namespace: vote
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
    nodePort: 31000
  selector:
    app: vote
  type: NodePort
```

### result-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: result
  name: result
  namespace: vote
spec:
  ports:
  - port: 8081
    protocol: TCP
    targetPort: 80
    nodePort: 31001
  selector:
    app: result
  type: NodePort
```

### Check services

```
➜  k get services -n vote
NAME     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
db       ClusterIP   172.20.104.164   <none>        5432/TCP         25m
redis    ClusterIP   172.20.102.135   <none>        6379/TCP         26m
result   NodePort    172.20.235.15    <none>        8081:31001/TCP   21m
vote     NodePort    172.20.224.132   <none>        8080:31000/TCP   29m
```
