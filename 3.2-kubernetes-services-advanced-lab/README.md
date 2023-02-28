## 3.2. Kubernetes Service - Advanced Services Lab

This is the 2nd of a series of labs about k8s services. This lab explores different scenarios for advertising services ip addresses via BGP.

In this lab, you will: 

* Advertise the service IP range
* Advertise individual service cluster IP
* Advertise individual service external IP

### Before you begin

This lab depends on the previous labs setup and requires Calico CNI set up, Yaobank application deployed, and BGP peering established with the bastion node. 

### Advertise service cluster IP addresses

Advertising services over BGP allows you to directly access the service without using NodePorts or a cluster Ingress Controller.


Before we begin, let's take a look at the state of routes on the bastion node.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
```

If you completed the previous lab correctly, you should see one route that was learned from Calico that provides access to the nginx pod that was created in the externally routable namespace (the route ending in `proto bird` in this example output). In this lab we will advertise Kubernetes services (rather than individual pods) over BGP.


A BGP configuration resource (BGPConfiguration) represents BGP specific configuration options for the cluster or a specific node. The resource with the name default has a specific meaning and contains the BGP global default configuration. By the default, there is no BGPConfiguration resource deployed in the cluster. However, Once BGP is activated, Calico has default built-in configurations for BGP in its data model. For example, it use 64512 as the default AS number.

Run the following and command and ensure that there is no bgpconfigurations in your cluster.

```
kubectl get bgpconfigurations
```

Examine the following default BGP configuration and then apply it.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: "10.49.0.0/16"
EOF

```
The `serviceClusterIPs` parameter tells Calico to advertise the cluster IP range.


Verify the BGPConfiguration just implemented.

```
kubectl get bgpconfigurations default -o yaml
```

```
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"projectcalico.org/v3","kind":"BGPConfiguration","metadata":{"annotations":{},"name":"default"},"spec":{"serviceClusterIPs":[{"cidr":"10.49.0.0/16"}]}}
  creationTimestamp: "2022-07-10T00:00:10Z"
  name: default
  resourceVersion: "64830"
  uid: 8dd4b681-84fe-4f39-b7e4-3f8384cb8ad0
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
```

Now that we have enabled Service ClusterIP advertisement, let's examine the routes on bastion node again.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
```

You should now see the cluster service cidr `10.49.0.0/16` advertised from each of the kubernetes cluster nodes. This means that traffic to any service's cluster IP address will get load-balanced across all nodes in the cluster by the network using ECMP (Equal Cost Multi Path). Kube-proxy then load balances the cluster IP across the service endpoints (backing pods) in exactly the same way as if a pod had accessed a service via a cluster IP from within the cluster.


Let's verify connectivity through the clusterIP. Find the cluster IP for the `customer` service.

```
kubectl get svc -n yaobank customer
```

```
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort   10.49.206.189   <none>        80:30180/TCP   119m
```

In this case, the ClusterIP is `10.49.206.189`. Your IP may be different.

Confirm you can access it from the bastion host.

```
curl 10.49.206.189
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


### Advertise individual service cluster IP

You can set `externalTrafficPolicy: Local` on a Kubernetes service to request that external traffic to a service only be routed via nodes which have a local service endpoint (backing pod). This preserves the client source IP and avoids the second hop when kube-proxy loadbalances to a service endpoint (backing pod) on another node. 

Traffic to the cluster IP for a service with `externalTrafficPolicy: Local` will be load-balanced across the nodes with endpoints for that service.
Note that `externalTrafficPolicy: Local` is only supported with service types of LoadBalancer and NodePort. For more information, visit the following link.

https://projectcalico.docs.tigera.io/networking/advertise-service-ips 


Update the `customer` service to add `externalTrafficPolicy: Local`.

```
kubectl patch svc -n yaobank customer -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

```
service/customer patched
```

Examine the routes on bastion node.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.49.206.189 via 10.0.1.30 dev ens5 proto bird 
```

You should now have a `/32` route for the yaobank customer service (`10.49.206.189` in the above example output) advertised from the node hosting the customer service pod (worker1, `10.0.1.30` in this example output).

```
kubectl get pods -n yaobank -l app=customer -o wide
```

```
NAME                        READY   STATUS    RESTARTS   AGE    IP          NODE                                      NOMINATED NODE   READINESS GATES
customer-68d67b588d-hn95n   1/1     Running   0          140m   10.48.0.8   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

For each active service with `externalTrafficPolicy: Local`, Calico advertise the IP for that service as a `/32` route from the nodes that have endpoints for that service. This means that external traffic to the service will get load-balanced across all nodes in the cluster that have a service endpoint (backing pod) for the service by the network using ECMP (Equal Cost Multi Path). Kube-proxy then DNATs the traffic to the local backing pod on that node (or load-balances equally to the local backing pods if there is more than one on the node).

The two main advantages of using `externalTrafficPolicy: Local` in this way are:
* There is a network efficiency win avoiding potential second hop of kube-proxy load-balancing to another node.
* The client source IP addresses are preserved, which can be useful if you want to restrict access to a service to specific IP addresses using network policy applied to the backing pods.


In the previous labs, we accessed the yaobank frontend UI using curl from the `control1` node and we also mentioned that we can try any other cluster node IP address to hit the NodePort. As we've now set `externalTrafficPolicy: Local`, this will no longer work since there are no `customer` pods hosted on `control1`. Accessing the NodePort can only happen via `worker1` at this point.

The following should fail.

```
curl 10.0.1.20:30180
```
The following should go through. Please make sure to use the node IP address where customer pod is running. 

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
```

### Advertise service external IP addresses

If you want to advertise a service using an IP address outside of the cluster service CIDR range, you can configure the service to have one or more `externalIPs`.


Before we begin, examine the kubernetes services in the `yaobank` kubernetes namespace.

```
kubectl get svc -n yaobank
```

```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.206.189   <none>        80:30180/TCP   153m
database   ClusterIP   10.49.130.110   <none>        2379/TCP       153m
summary    ClusterIP   10.49.198.166   <none>        80/TCP         153m
```

Note that none of them currently have an `EXTERNAL-IP`.
Update the Calico BGPconfiguration to advertise a service external IP CIDR range of `10.50.0.0/24`.

```
kubectl patch bgpconfigurations default --patch    '{"spec": {"serviceExternalIPs": [{"cidr": "10.50.0.0/24"}]}}'
```

```
bgpconfiguration.projectcalico.org/default patched
```

Validate the BGPconfiguration applied.

```
kubectl get bgpconfigurations default -o yaml
```

```
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"projectcalico.org/v3","kind":"BGPConfiguration","metadata":{"annotations":{},"name":"default"},"spec":{"serviceClusterIPs":[{"cidr":"10.49.0.0/16"}]}}
  creationTimestamp: "2022-07-10T00:00:10Z"
  name: default
  resourceVersion: "71267"
  uid: 8dd4b681-84fe-4f39-b7e4-3f8384cb8ad0
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
  serviceExternalIPs:
  - cidr: 10.50.0.0/24
```


Note that `serviceExternalIPs` is a list of CIDRs. You could, for example, add individual /32 IP addresses if there were just a small number of specific IPs you wanted to advertise.


Examine the routes on bastion node.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.49.206.189 via 10.0.1.30 dev ens5 proto bird 
10.50.0.0/24 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
```

You should now have a route for the external ID CIDR (10.50.0.10/24) with next hops to each of our cluster nodes.

Assign the service external IP `10.50.0.10` to the `customer` service.

```
kubectl patch svc -n yaobank customer -p  '{"spec": {"externalIPs": ["10.50.0.10"]}}'
```
```
service/customer patched
```

Examine the services again to validate everything is as expected.

```
kubectl get svc -n yaobank
```

```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.206.189   10.50.0.10    80:30180/TCP   161m
database   ClusterIP   10.49.130.110   <none>        2379/TCP       161m
summary    ClusterIP   10.49.198.166   <none>        80/TCP         161m

```

You should now see the external ip (`10.50.0.10`) assigned to the `customer` service.  We can now access the `customer` service from outside the cluster using the external ip address (10.50.0.10) we just assigned.

Connect to the `customer` service from the bastion node using the service external IP `10.50.0.10`.

```
curl 10.50.0.10
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
</html>
```

As you can see, the service has been made available outside of the cluster via bgp routing and load balancing.

#### Recap

We've covered five different ways so far in our lab for connecting to your pods from outside the cluster.
* Via a standard NodePort on a specific node. (This is how you connected to the YAO Bank web frontend)
* Direct to the pod IP address by configuring a Calico IP Pool that is externally routable.
* Advertising the service cluster IP range. (And using ECMP to load balance across all nodes in the cluster)
* Advertising individual cluster IPs. (Services with `externalTrafficPolicy: Local`, using ECMP to load balance only to the nodes hosting the pods backing the service)
* Advertising service externalIPs. (So you can use service IP addresses outside of the cluster IP range)

There are more ways for cluster external connectivity, nonetheless we have covered the most common scenarios. This gives you an idea about Calico's versatility and ability to fit with a broad range of networking needs.

> __Congratulation! You have completed this lab and you have by now a good understanding of k8s services external connectivity__

