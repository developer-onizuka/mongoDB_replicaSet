# mongoDB_replicaSet

# 0. Requirements
- Create the k8s cluster in which pods can access to other virtual machines. See https://github.com/developer-onizuka/iptables_SNAT

- Create the Ops Manager and mongoDB replicaSets. See https://github.com/developer-onizuka/mongoDB_opsManager

| Virtual Machine | IPaddress | Function |
| --- | --- | --- |
| Master | 192.168.33.100 | k8s's master |
| Worker1 | 192.168.33.101 | k8s's worker1 |
| Worker2 | 192.168.33.102 | k8s's worker2 |
| mongo-0 | 192.168.33.31 | mongoDB replicaSet |
| mongo-1 | 192.168.33.32 | mongoDB replicaSet |
| mongo-2 | 192.168.33.33 | mongoDB replicaSet |
| ops-manager | 192.168.33.12 | mongoDB ops Manager |


# 1. Create deployment of "Employee Web app" with 4 repricas awaring mongoDB's ReplicaSet
```
$ git clone https://github.com/developer-onizuka/mongoDB_replicaSet.git
$ cd mongoDB_replicaSet
$ sudo kubectl apply -f employee-replica-opsmanager.yaml 
service/employee-srv unchanged
deployment.apps/employee-test configured
```

You should change the value among 192.168.33.30:27017, 192.168.33.31:27017 or 192.168.33.32:27017, depending on the Primary of MongoDB.
```
        env:
        - name: MONGO
          #value: mongo-srv
          #value: mongo-test-0.mongo-srv,mongo-test-1.mongo-srv,mongo-test-2.mongo-srv
          value: 192.168.33.30:27017
```

# 2. Create Nginx's config files and Configmap
```
$ sudo kubectl create configmap nginx-config --from-file=default.conf
configmap/nginx-config created
```

# 3. Create depolyment of Nginx with 2 repricas
```
$ sudo kubectl apply -f nginx-nodeport.yaml
```

# 4. Check if pods are available
```
$ sudo kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP                NODE      NOMINATED NODE   READINESS GATES
curl                             1/1     Running   0          24m     192.168.189.65    worker2   <none>           <none>
employee-test-58c7c99ff4-269fk   1/1     Running   0          92s     192.168.235.133   worker1   <none>           <none>
employee-test-58c7c99ff4-k4klg   1/1     Running   0          94s     192.168.189.69    worker2   <none>           <none>
employee-test-58c7c99ff4-ml9kg   1/1     Running   0          94s     192.168.235.132   worker1   <none>           <none>
employee-test-58c7c99ff4-qdq2z   1/1     Running   0          92s     192.168.235.134   worker1   <none>           <none>
nginx-test-67d9db6c48-2gwnb      1/1     Running   0          7m58s   192.168.189.67    worker2   <none>           <none>
nginx-test-67d9db6c48-tw2ln      1/1     Running   0          7m58s   192.168.189.68    worker2   <none>           <none>

$ sudo kubectl get services -o wide
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE   SELECTOR
employee-srv   ClusterIP   10.109.161.133   <none>        5001/TCP,5000/TCP   51m   run=employee-test
kubernetes     ClusterIP   10.96.0.1        <none>        443/TCP             88m   <none>
nginx-srv      NodePort    10.100.42.100    <none>        8080:30001/TCP      44m   run=nginx-test
```

# 5. Access to each nodeport
In this case the nodeport is 192.168.33.100:30001, 192.168.33.101:30001 or 192.168.33.102:30001.

https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/Screenshot%20from%202021-10-15%2022-14-37.png
https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/Screenshot%20from%202021-10-15%2021-31-17.png
