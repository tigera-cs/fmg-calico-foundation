## 4.1. Policy: Basic Calico Network Policy Lab
This is the first lab of a series of labs focusing on Calico k8s network policy. Throughout this lab we will deploy and test our first Calico k8s network policy. 

In this lab we will:
* Apply a simple Calico Policy


### Apply a simple Calico Policy

Deploy yaobank application.

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

Let's start by finding the IP address of the summary service and pods.

```
kubectl get pod -n yaobank -l app=summary -o wide
```

```
NAME                       READY   STATUS    RESTARTS   AGE     IP            NODE                                      NOMINATED NODE   READINESS GATES
summary-748b977d44-6m6gd   1/1     Running   0          2m48s   10.48.0.198   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
summary-748b977d44-hmkq7   1/1     Running   0          2m48s   10.48.0.13    ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

```
kubectl get svc -n yaobank -l app=summary -o wide
```

```
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
summary   ClusterIP   10.49.202.152   <none>        80/TCP    3m59s   app=summary
```

Next, Let's exec into the customer pod and perform the following connectivity checks. Please note that the pod IP address could be different in your lab.

* Ping the summary pod IP
* Curl the summary pod IP
* Ping summary (service name)
* Curl summary (service name)

```
 kubectl exec -ti -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) -- bash
```

```
ping 10.48.0.198
```

```
curl -v telnet://10.48.0.198:80
```

```
ping summary
```

```
curl -v telnet://summary:80
```

Exit the customer pod.

```
exit
```

You should have the following behaviour:

* Ping to the pod IP is successful
* Curl to the pod IP is successful
* Ping to summary fails
* Curl to summary is successful

As we have learned in Lab3, services are serviced by kube-proxy, which load-balances the service request to backing pods. The service is listening to TCP port 80 so the ping failure is expected. Service IP addresses are virtual IP addresses and are not pingable.

Verify external access connectivity from your bastion node to the customer service `NodePort:30180`.

```
curl 10.0.1.30:30180
```

```
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title>YAO Bank</title>
    <style>
    h2 {
      font-family: Arial, Helvetica, sans-serif;
    }
    h1 {
      font-family: Arial, Helvetica, sans-serif;
    }
    p {
      font-family: Arial, Helvetica, sans-serif;
    }
    </style>
  </head>
  <body>
        <h1>Welcome to YAO Bank</h1>
        <h2>Name: Spike Curtis</h2>
        <h2>Balance: 2389.45</h2>
        <p><a href="/logout">Log Out >></a></p>
  </body>
</html
```


Let's limit connectivity on the summary pods to only allow inbound traffic to port 80 from the customer pod.
Examine the manifest before applying it.

``` 
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: customer2summary
  namespace: yaobank
spec:
  order: 500
  selector: app == "summary"
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "customer"
    destination:
      ports:
      - 80
  types:
    - Ingress
EOF

```

The policy applies to pods, which has the label app=summary in the yaobank namespace. The policy allows TCP port 80 traffic from the source matching the label app=customer in the yaobank namespace.


Now, let's repeat the tests we did before.

```
 kubectl exec -ti -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) -- bash
```

```
ping 10.48.0.198
```

```
curl -v telnet://10.48.0.198:80
```

```
curl -v telnet://summary:80
```

Exit the customer pod.

```
exit
```

You should have the following behaviour:

* Ping to the pod IP now fails. This is expected since icmp was not allowed in the policy we have applied.
* Curl to the pod IP is successful
* Curl to summary is successful

Let's cleanup the network policy for now.

```
kubectl delete -n yaobank networkpolicy.pro customer2summary
```
> __Congratulations! You have completed your first Calico network policy lab.__
