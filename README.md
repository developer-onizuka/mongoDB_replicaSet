# mongoDB_replicaSet

https://www.youtube.com/watch?v=7howpsD9Ogc

# 0. Prerequisites and what you must create before trying mongoDB's replicaSet

| Virtual Machine | IPaddress | Function | Vagrantfile |
| --- | --- | --- | --- |
| Master | 192.168.33.100 | k8s's master | [Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/Vagrantfile1) |
| Worker1 | 192.168.33.101 | k8s's worker1 |^|
| Worker8 | 192.168.33.108 | k8s's worker8 | [Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/Vagrantfile2) |
| Worker9 | 192.168.33.109 | k8s's worker9 |^|
| mongo-0 | 192.168.33.31 | mongoDB replicaSet | [Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/Vagrantfile3) |
| mongo-1 | 192.168.33.32 | mongoDB replicaSet |^|
| mongo-2 | 192.168.33.33 | mongoDB replicaSet |^|
| ops-manager | 192.168.33.12 | mongoDB ops Manager | [Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/Vagrantfile4) |

- Create kubernetes cluster attached LoadBalancer such as MetalLB. You might use the Vagrantfiles above and the link below for MetalLB system:
  > https://github.com/developer-onizuka/metalLB

- Download and install istio and make label on the default namespace with istio-injection=enabled.
  > https://github.com/developer-onizuka/istio

- Create Three Virtual Machines for mongoDB's replicaSets which will be created later. 
- Create the Ops Manager. 
  > https://github.com/developer-onizuka/mongoDB_opsManager

- Istio's WorkloadEntry creates a kind of resolvor between endpoint and IP address for the pod in the Service mesh. You don't need creating entries in /etc/hosts as like below if you create WorkloadEntry resource thru mongo-vm-svc.yaml and mongo-vm-wkle.yaml attached.
```
192.168.33.30 mongo-0
192.168.33.31 mongo-1
192.168.33.32 mongo-2
```

# 1. Create ReplicaSet among mongo-0, mongo-1 and mongo-2 with mongoDB opsManager
  > https://github.com/developer-onizuka/mongoDB_opsManager#4-mongodb-mms-agent-startup-daemon

# 2. Create Services and workloadEntries bound for mongoDB's replicaSet outside of kubernetes cluster
```
$ git clone https://github.com/developer-onizuka/mongoDB_replicaSet.git
$ cd mongoDB_replicaSet
$ kubectl apply -f mongo-vm-svc.yaml,mongo-vm-wkle.yaml 
service/mongo-0 created
service/mongo-1 created
service/mongo-2 created
workloadentry.networking.istio.io/mongo-0-vm-wkle created
workloadentry.networking.istio.io/mongo-1-vm-wkle created
workloadentry.networking.istio.io/mongo-2-vm-wkle created
```

# 3. Create deployment of "Employee Web app" with 2 repricas awaring mongoDB's ReplicaSet
```
$ kubectl apply -f employee-replica-opsmanager.yaml 
service/employee-svc created
deployment.apps/employee-test created
```

The environment of MONGO is for setting a connection string, but in this case it is a replicaSet aware connection string, especially.
```
        env:
        - name: MONGO
          #value: 'mongo-0'
          value: 'mongo-0:27017,mongo-1:27017,mongo-2:27017/?replicaSet=myReplicaSet'
          #value: 192.168.33.30:27017,192.168.33.31:27017,192.168.33.32:27017/?replicaSet=myReplicaSet
```

# 4. Create Ingress Gateway for accessing from outside of the Cluster
```
$ kubectl apply -f ingress-gateway.yaml 
gateway.networking.istio.io/employee-gateway created
```

# 5. Create Nginx's config files and Configmap
```
$ kubectl create configmap nginx-config --from-file=default.conf
configmap/nginx-config created
```

# 6. Create depolyment of Nginx with 2 repricas
```
$ kubectl apply -f nginx.yaml
```

# 7. Check if all of pods are available
```
$ kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE    IP              NODE      NOMINATED NODE   READINESS GATES
employee-test-59d8ff8d6d-gjch9   2/2     Running   0          171m   10.10.45.251    worker8   <none>           <none>
employee-test-59d8ff8d6d-pxxbp   2/2     Running   0          171m   10.10.235.158   worker1   <none>           <none>
nginx-test-57c5b8b58d-8mc9q      2/2     Running   0          5s     10.10.235.132   worker1   <none>           <none>
nginx-test-57c5b8b58d-s9fj4      2/2     Running   0          5s     10.10.215.32    worker9   <none>           <none>
```

# 8. Check services and workloadEntries
```
$ kubectl get services -o wide
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE    SELECTOR
employee-svc   ClusterIP   10.99.68.147     <none>        5001/TCP,5000/TCP   172m   app=employee-test
kubernetes     ClusterIP   10.96.0.1        <none>        443/TCP             16d    <none>
mongo-0        ClusterIP   10.111.158.28    <none>        27017/TCP           170m   app=mongo-0-vm
mongo-1        ClusterIP   10.105.199.217   <none>        27017/TCP           170m   app=mongo-1-vm
mongo-2        ClusterIP   10.97.52.44      <none>        27017/TCP           170m   app=mongo-2-vm
nginx-svc      ClusterIP   10.100.134.248   <none>        8080/TCP            61s    app=nginx-test
```
```
$ kubectl get workloadentry
NAME              AGE     ADDRESS
mongo-0-vm-wkle   8m11s   192.168.33.30
mongo-1-vm-wkle   8m11s   192.168.33.31
mongo-2-vm-wkle   8m11s   192.168.33.32
```
```
$ kubectl get endpoints
NAME           ENDPOINTS                                                             AGE
employee-svc   10.10.235.158:5001,10.10.45.251:5001,10.10.235.158:5000 + 1 more...   172m
kubernetes     192.168.33.100:6443                                                   16d
mongo-0        <none>                                                                170m
mongo-1        <none>                                                                170m
mongo-2        <none>                                                                170m
nginx-svc      10.10.215.32:80,10.10.235.132:80                                      81s
```

# 9. Let's Access to it 

Find the IP address of Istio-ingressgateway. In this case, it is 192.168.33.220.
```
kubectl get services -n istio-system 
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                           AGE
grafana                 ClusterIP      10.103.59.87     <none>           3000/TCP                                                          15d
istio-eastwestgateway   LoadBalancer   10.109.178.196   192.168.33.221   15021:30600/TCP,15443:31534/TCP,15012:31242/TCP,15017:30426/TCP   16d
istio-ingressgateway    LoadBalancer   10.110.212.70    192.168.33.220   15021:31932/TCP,80:30217/TCP,443:31930/TCP                        16d
istiod                  ClusterIP      10.111.13.175    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                             16d
jaeger-collector        ClusterIP      10.101.9.249     <none>           14268/TCP,14250/TCP,9411/TCP                                      15d
kiali                   LoadBalancer   10.107.184.95    192.168.33.222   20001:32532/TCP,9090:32092/TCP                                    15d
prometheus              ClusterIP      10.109.69.225    <none>           9090/TCP                                                          15d
tracing                 ClusterIP      10.111.142.142   <none>           80/TCP,16685/TCP                                                  15d
zipkin                  ClusterIP      10.101.75.155    <none>           9411/TCP  
```
If you find such as below, you succeed deploying mongoDB's replicaSet.

![mongoDB-replicaSet1.png](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/mongoDB-replicaSet1.png)


But if you find the following message while accessing to the replicaSet, it means the Appication could not resolve the hostname of mongo-0, mongo-1 and mongo-2. Check if the workloadEntry is created properly.
---
```
$ curl https://localhost:5001 -k
System.TimeoutException: A timeout occurred after 30000ms selecting a server using CompositeServerSelector{ Selectors = MongoDB.Driver.MongoClient+AreSessionsSupportedServerSelector, 
LatencyLimitingServerSelector{ AllowedLatencyRange = 00:00:00.0150000 }, OperationsCountServerSelector }. Client view of cluster state is { ClusterId : "1", 
ConnectionMode : "ReplicaSet", Type : "ReplicaSet", State : "Disconnected", Servers : [{ ServerId: "{ ClusterId : 1, EndPoint : "Unspecified/mongo-0:27017" }", 
EndPoint: "Unspecified/mongo-0:27017", ReasonChanged: "Heartbeat", State: "Disconnected", ServerVersion: , TopologyVersion: , Type: "Unknown", 
HeartbeatException: "MongoDB.Driver.MongoConnectionException: An exception occurred while opening a connection to the server.
```



You also find the access routes seeing kiali which is an addon process of istio.

![mongoDB-replicaSet2.png](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/mongoDB-replicaSet2.png)


# X. mongoDB compass
The mongoDB compass is a nice tool. Download and use it to see DB inside.
- https://www.mongodb.com/ja-jp/products/compass
