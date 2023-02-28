## 4.3. Network Policy - Advanced Lab

This is the 3rd lab in a series of labs exploring network policies.

In this lab you will:

* Create Egress Lockdown policy as a Security Admin for the cluster
* Grant selective Internet access
* Apply Calico network policy based on Kubernetes services


### Before you begin

This lab builds on top of the previous labs. Please make sure you have completed the previous labs before starting this lab.


### Create Egress Lockdown policy as a Security Admin for the cluster

This lab guides you through the process of deploying a egress lockdown policy in our cluster.
In the previous lab, we applied a policy that allows wide open egress access to customer and summary pods. Best-practices call for a restrictive policy that allows minimal access and denies everything else.

Let's first start with verifying connectivity with the configuration applied in the previous lab. Confirm that the customer pod is able to initiate connections to the Internet.

Exec into the customer pod.

```
CUSTOMER_POD=$(kubectl get pod -l app=customer -n yaobank -o jsonpath='{.items[0].metadata.name}')
echo $CUSTOMER_POD
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash

```

```
ping -c 3 8.8.8.8
```

```
curl -I www.google.com
```

```
exit
```

This succeeds since the policy in place allows full internet access.

Let's now create a Calico GlobalNetworkPolicy to restrict egress access to the Internet to only pods that have the ServiceAccount that is labeled  "internet-egress = allowed".


Examine the policy before applying it. While Kubernetes network policies only have Allow rules, Calico network policies also support Deny rules. As this policy has Deny rules in it, it is important that we set its precedence higher than the K8s policy Allow rules. To do this, we specify `order`  value of 600 in this policy, which gives it higher precedence than the k8s policy (which does not have the concept of policy precedence, and is assigned a fixed order value of 1000 by Calico). 

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: egress-lockdown
spec:
  order: 600
  selector: ''
  types:
  - Egress
  egress:
    - action: Allow
      source:
        serviceAccounts:
          selector: internet-egress == "allowed"
      destination: {}
    - action: Deny
      source: {}
      destination:
        notNets:
          - 10.48.0.0/16
          - 10.49.0.0/16
          - 10.50.0.0/24
          - 10.0.0.0/24
EOF

```
Notice the notNets destination parameter that excludes known cluster networks from the deny rule. 

Verify Internet access again. Exe into the customer pod.

```
CUSTOMER_POD=$(kubectl get pod -l app=customer -n yaobank -o jsonpath='{.items[0].metadata.name}')
echo $CUSTOMER_POD
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash

```

```
ping -c 3 8.8.8.8
```

```
curl -I www.google.com
```

```
exit
```

These commands should fail. Pods are now restricted to only accessing other pods and nodes within the cluster. You may need to terminate the command with ctrl+c and exit back to your node.

```
exit
```

### Grant selective Internet access

Now let's take the case where there is a legitimate reason to allow connections from the Customer pod to the Internet. As we used a Service Account label selector in our egress policy rules, we can enable this by adding the appropriate label to the pod's Service Account.


```
kubectl label serviceaccount -n yaobank customer internet-egress=allowed
```

Verify that the customer pod can now access the Internet.

```
CUSTOMER_POD=$(kubectl get pod -l app=customer -n yaobank -o jsonpath='{.items[0].metadata.name}')
echo $CUSTOMER_POD
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash

```

```
ping -c 3 8.8.8.8
```

```
curl -I www.google.com
```

```
exit
```

Now you should find that the customer pod is allowed Internet access, but other pods (like Summary and Database) are not.


There are many ways for dividing responsibilities across teams using Kubernetes RBAC.

Let's take the following case:

* The SecOps team is responsible for creating Namespaces and Services accounts for dev teams. Kubernetes RBAC is setup so that only they can do this.
* Dev teams are given Kubernetes RBAC permissions to create pods in their Namespaces and they can use but not modify any Service Account in their Namespaces.

In this scenario, the SecOps team can control which teams should be allowed to have pods that access the Internet.  If a dev team is allowed to have pods that access the Internet then the dev team can choose which pods access the Internet by using the appropriate Service Account. 

This is just one way of dividing responsibilities across teams.  Pods, Namespaces, and Service Accounts all have separate Kubernetes RBAC controls and they can all be used to select workloads in Calico network policies.

### Apply Calico network policy based on Kubernetes services

Calico and Kubernetes policies are implemented based on endpoint labels. However, Calico provides a convenient way of defining policies based on Kubernetes Services and Calico dynamically in the backend implements and enforces the policy on the associated endpoints.

Let's start by creating an application that is composed of a server and a client component. Get yourself familiar with the following app. We will use nginx-client to establish http connections to nginx-server on port 80.

```
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nginxapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-client
  namespace: nginxapp
  labels:
    app: nginx-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-client
  template:
    metadata:
      name: nginx-client
      labels:
        app: nginx-client
    spec:
      containers:
      - image: praqma/network-multitool
        imagePullPolicy: IfNotPresent
        name: multitool
        ports:
        - containerPort: 80
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-client
  name: nginx-client
  namespace: nginxapp
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-client
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  namespace: nginxapp
  labels:
    app: nginx-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-server
  template:
    metadata:
      labels:
        app: nginx-server
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-server
  name: nginx-server
  namespace: nginxapp
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-server
EOF

```

```
kubectl get pods -n nginxapp -o wide
```

```
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE                                      NOMINATED NODE   READINESS GATES
nginx-client-5444bb9bb9-6gt92   1/1     Running   0          70m   10.48.0.199   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
nginx-server-6f6f496597-9jcd5   1/1     Running   0          58m   10.48.0.200   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
nginx-server-6f6f496597-dzfvp   1/1     Running   0          58m   10.48.0.17    ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
nginx-server-6f6f496597-fxlrr   1/1     Running   0          58m   10.48.0.16    ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

```
kubectl get svc -n nginxapp
```

```
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-client   ClusterIP   10.49.23.129   <none>        80/TCP    58m
nginx-server   ClusterIP   10.49.191.55   <none>        80/TCP    56m
```

Let's exec into the nginx-client pod and try connecting to nginx-server service.

```
nginx_client_pod=$(kubectl get pods -n nginxapp -l app=nginx-client -o jsonpath='{.items[0].metadata.name}')
echo $nginx_client_pod
kubectl exec -ti $nginx_client_pod -n nginxapp -- bash
```
The connection should fail as our previously implemented global network policy is blocking the traffic. Please note that the service IP address could be different for you.

```
curl 10.49.191.55
```

Let's now implement the required Calico network policies to enable the communication. Examine the following Calico network policies and then apply them.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: nginxclient-allow
  namespace: nginxapp
spec:
  selector: app == "nginx-client"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        services:
          name: nginx-server
          namespace: nginxapp
      destination: {}
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        services:
          name: nginx-server
          namespace: nginxapp
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: nginxserver-allow
  namespace: nginxapp
spec:
  selector: app == "nginx-server"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        services:
          name: nginx-client
          namespace: nginxapp
      destination:
        ports:
          - '80'
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        services:
          name: nginx-client
          namespace: nginxapp
  types:
    - Ingress
    - Egress
EOF

```
Exec into the nginx-client pod and try connecting to nginx-server service again.

```
nginx_client_pod=$(kubectl get pods -n nginxapp -l app=nginx-client -o jsonpath='{.items[0].metadata.name}')
echo $nginx_client_pod
kubectl exec -ti $nginx_client_pod -n nginxapp -- bash
```
This time the connection should succeed. Please note that the service IP address could be different for you.

```
curl 10.49.191.55
```


> __Congratulations! You have completed your Calico advanced policy lab.__
