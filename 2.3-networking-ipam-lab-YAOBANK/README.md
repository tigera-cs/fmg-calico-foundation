## 2.3. Networking: Calico-IPAM Lab

This is the 3rd lab in a series of labs exploring k8s networking. It explores k8s ip adress management via Calico IPAM.

In this lab, you will:

* Check existing IPPools and create a new IPPool
* Update the yaobank deployments to receive IP addresses from the new IPPool


### Check existing IPPools and create a new IPPool

Check the IPPools that exist in the cluster. You should see two IPPools, one (default-ipv4-ippool) was created at the cluster creation time using the Installation resource and the other (external-pool) was created in the previous lab as part of checking pod's external connectivity exercise.

```
kubectl get ippools.projectcalico.org
```
You should receive an output similar to the following.
```
NAME                  CREATED AT
default-ipv4-ippool   2022-07-09T17:44:37Z
external-pool         2022-07-09T20:02:25Z
```
Let's check the details of the `default-ipv4-ippool` and get familiar with some configuration parameters in the IPPool.

```
kubectl get ippool default-ipv4-ippool -o yaml
```

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-07-09T17:44:37Z"
  name: default-ipv4-ippool
  resourceVersion: "7202"
  uid: 49ecd163-1a83-4566-872a-fb0390102724
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.48.0.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```

We have extracted the relevant information here. You can see from the output that the default IPPool range is 10.48.0.0/24, which is actually the Calico Initial default IP Pool range of our k8s cluster.
Note the relevant information in the manifest:

* allowedUses: specifies if this IPPool can be used for tunnel interfaces, workload interfaces, or both.
* blockSize: used by Calico IPAM to efficiently assign IPAM Blocks to node and advertise them between different nodes.
* cidr: specifies the IP range to be used for this IPPool.
* ipipMode/vxlanMode: used to enable or disable ipip and vxlan overlay. options are never, always and crosssubnet.
* natOutgoing: specifies if SNAT should happen when pods try to connect to destination outside the cluster. `natOutgoing` must be set to `true` when using overlay networking mode for external connectivity.
* nodeSelector: selects the nodes that Calico IPAM should assign addresses from this pool to.


Let's create a new IPPool by applying the following manifest. This time we want to use this IPPool to assign IP addresses to specific deployments instead of all the pods deployed in a namespace.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool2-ipv4-ippool
spec:
  blockSize: 26
  cidr: 10.48.128.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
EOF

```
You should receive an output similar to the following.

```
ippool.projectcalico.org/pool2-ipv4-ippool created
```
Check the IPPool that exist in the cluster.

```
calicoctl get ippools
```
```
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   10.48.0.0/24     all()      
external-pool         10.48.2.0/24     all()      
pool2-ipv4-ippool     10.48.128.0/24   all()  
```


### Update the yaobank deployments to receive IP addresses from the new IPPool

There is a new version of the yaobank application that is configured for specific IP address treatment. We have configured the yaobank manifest with the necessary annotation to use the newly created IPPool. Note that annotations are configured only for summary and database deployment. Customer deployment can receive its IP address from any IPPool. Examine the annotation section of the deployments below and get yourself familiar with the configurations. 

Before deploying the new version of yaobank application, let's delete the old version to avoid any conflicts.

```
kubectl delete namespace yaobank
```
You should receive an output similar to the following. This command might take about 1-2 minutes to complete. Please wait util the namespace and all of the resources assocaited with it are deleted.

```
namespace "yaobank" deleted
```
Now implement the new version of the app.

```
kubectl apply -f -<<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: yaobank
  labels:
    istio-injection: disabled

---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: yaobank
  labels:
    app: database
spec:
  ports:
  - port: 2379
    name: http
  selector:
    app: database

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database
  namespace: yaobank
  labels:
    app: yaobank

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: yaobank
spec:
  selector:
    matchLabels:
      app: database
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: database
        version: v1
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"
    spec:
      serviceAccountName: database
      containers:
      - name: database
        image: calico/yaobank-database:certification
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 2379
        command: ["etcd"]
        args:
          - "-advertise-client-urls"
          - "http://database:2379"
          - "-listen-client-urls"
          - "http://0.0.0.0:2379"

---
apiVersion: v1
kind: Service
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: summary
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: summary
    
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: yaobank
    database: reader
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: summary
  namespace: yaobank
spec:
  replicas: 2
  selector:
    matchLabels:
      app: summary
      version: v1
  template:
    metadata:
      labels:
        app: summary
        version: v1
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"
    spec:
      serviceAccountName: summary
      containers:
      - name: summary
        image: calico/yaobank-summary:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
 
---
apiVersion: v1
kind: Service
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: customer
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30180
    name: http
  selector:
    app: customer
    
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: yaobank
    summary: reader
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer
  namespace: yaobank
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customer
      version: v1
  template:
    metadata:
      labels:
        app: customer
        version: v1
    spec:
      serviceAccountName: customer
      containers:
      - name: customer
        image: calico/yaobank-customer:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
EOF

```

You should receive an output similar to the following.

```
namespace/yaobank created
service/database created
serviceaccount/database created
deployment.apps/database created
service/summary created
serviceaccount/summary created
deployment.apps/summary created
service/customer created
serviceaccount/customer created
deployment.apps/customer created
```

Let's check on the Pods' ip address assignments.

```
kubectl get pod -n yaobank -o wide
```

```
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE                                      NOMINATED NODE   READINESS GATES
customer-68d67b588d-hn95n   1/1     Running   0          63s   10.48.0.8       ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
database-769f6644c5-t925v   1/1     Running   0          64s   10.48.128.0     ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
summary-dc858dd7b-mt5gv     1/1     Running   0          64s   10.48.128.1     ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
summary-dc858dd7b-pkt6l     1/1     Running   0          63s   10.48.128.192   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>

```

You can see that Pod IP address assignment is aligned with the IPAM configurations defined in manifest above, assigning pods of a deployment to the correct IPPool. Calico IPAM provides the flexibility of assigning IPPools to namespaces, deployment, deamonset, etc. You can also implement topology based IP address assignment in which racks or nodes in specific racks receive their IP addresses from one or more specific IPPools. For more information, visit the following link.

https://projectcalico.docs.tigera.io/networking/assign-ip-addresses-topology


> __Congratulations! You have completed you Calico IPAM lab.__ 
