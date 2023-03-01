## 2.2. Networking:  Pod Connectivity - Advanced Lab

This is the 2nd lab in a series of labs exploring k8s networking. This lab is focused on understanding relevant address ranges and BGP advertisement.

In this lab, you will:

* Examine IP address ranges used by the cluster
* Create an additional Calico IPPool
* Configure Calico BGP Peering to connect with a network outside of the cluster
* Configure a namespace to use externally routable IP addresses

### Before you begin

Please make sure you have completed the previous labs before starting this lab. You should have deployed Calico and the Yaobank sample application in your cluster. 


### Examine IP address ranges used by the cluster


When a Kubernetes cluster is bootstrapped, there are two address ranges that are configured. It is very important to understand these address ranges as they can't be changed once the cluster is created.

* The cluster pod network CIDR is the range of IP addresses Kubernetes is expecting to be assigned to the pods in the cluster.
* The services CIDR is the range of IP addresses that are used for the Cluster IPs of Kubernetes Sevices (the virtual IP that corresponds to each Kubernetes Service).

These are configured at cluster creation time (e.g. as initial kubeadm configuration).

You can find these values using the following command.

```
kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
```

```
kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
                            "--service-cluster-ip-range=10.49.0.0/16",
                            "--cluster-cidr=10.48.0.0/16",
```


### Create an additional Calico IPPool

When Calico is deployed, a defaul IPPool is created in the cluster for each address family (IPv4-IPv6) enabled in the cluster. This cluster runs only IPv4. As a result, we will only have an IPPool for IPv4. By default, Calico creates a default IPPool for the whole cluster pod network CIDR range. However, this can be customized and a subset of pod network CIDR can be used for the default IPPool.\
Let's find the configured IPPool in this cluster using the following command.

```
calicoctl get ippools
```

```
NAME                  CIDR           SELECTOR   
default-ipv4-ippool   10.48.0.0/24   all() 
```

Please note that you can also get IPPool information using `kubectl` instead of `calicoctl` in the previous command. If you use Openshift, you can replace `calicoctl` with `oc`.

In this cluster, Calico has been configured to allocate IP addresses for pods from the `10.48.0.0/24` CIDR (which is a subset of the `10.48.0.0/16` configured on Kubernetes).

We have the following address ranges configured in this cluster.

| CIDR         |  Purpose                                                  |
|--------------|-----------------------------------------------------------|
| 10.48.0.0/16 | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24 | Calico - Initial default IPPool                           |
| 10.49.0.0/16 | Kubernetes Service Network (via kubeadm `--service-cidr`) |



Calico provides a sophisticated and powerful IPAM solution, which enables you to allocate and manage IP addresses for a variety of use cases and requirements.

One of the use cases of Calico IPPool is to distinguish between different ranges of IP addresses that have different routablity scopes. If you are operating at a large scale, then IP addresses are precious resources. You might want to have a range of IPs that is only routable within the cluster, and another range of IPs that is routable across your enterprise. In that case, you can choose which pods get IPs from which range depending on whether workloads from outside of the cluster need direct access to the pods or not.

We'll simulate this use case in this lab by creating a second IPPool to represent the externally routable pool.  (And we've already configured the underlying network to not allow routing of the existing IPPool outside of the cluster.)


We're going to create a new pool for `10.48.2.0/24` that is externally routable.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: external-pool
spec:
  cidr: 10.48.2.0/24
  blockSize: 29
  ipipMode: Never
  natOutgoing: true
EOF

```
```
calicoctl get ippools
```
```
NAME                  CIDR           SELECTOR   
default-ipv4-ippool   10.48.0.0/24   all()      
external-pool         10.48.2.0/24   all()       
```

We now have the followings:

| CIDR         |  Purpose                                                  |
|--------------|-----------------------------------------------------------|
| 10.48.0.0/16 | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24 | Calico - Initial default IPPool                          |
| 10.48.2.0/24 | Calico - External IPPool (externally routable)           |
| 10.49.0.0/16 | Kubernetes Service Network (via kubeadm `--service-cidr`) |

### Configure Calico BGP Peering to connect with a network outside of the cluster

Let's start by examining Calico BGP peering status on one of the nodes. There are different methods to find BGP status information, but all these methods require access to `calicoctl` in some form. The reason for this is that bird requires privileged access to the local bird socket to provide status information. In this excerise, we are using `calicoctl` configured as a binary on the node.

SSH into worker1.

```
ssh worker1
```
Download `calicoctl` binary and make in executable. Please note that `calicoctl` uses Kubernetes kubeconfig file to authenticate to the cluster and run commands against the Kubernetes API. The kubeconfig file is already configured for you.

```
curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl
chmod +x calicoctl
sudo mv calicoctl /usr/local/bin

```


Check the BGP connection status from worker1 to other nodes (BGPPeers) in the cluster.

```
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.1.20    | node-to-node mesh | up    | 17:44:47 | Established |
| 10.0.1.31    | node-to-node mesh | up    | 17:44:46 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```
This above output shows that currently Calico on worker1 is only peering with the other nodes (control1 and worker2) in the cluster and is not peering with any router outside of the cluster.

```
exit
```


Now, let's simulate BGP peering to a router outside of the cluster by peering to bastion node. We've already set up bastion node to act as a router and it is ready to accept new BGP peering.

If you are interested to see the standalone bird configurations on `bastion` node, run the following command from the `bastion` node.

```
sudo cat /etc/bird/bird.conf
```

Let's add a new BGPPeer by running the following command. Get yourself familiar with the GBPPeer resource. `peerIP` is the IP address of the peering router, which is the `bastion` node in this case. `asNumber` is the AS number of the peering router.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
EOF

```
You should receive an output similar to the following.

```
bgppeer.projectcalico.org/bgppeer-global-64512 created
```

Let's examine the BGP peering from worker1 node again by doing an SSH into the node.

```
ssh worker1
```

Check the status of Calico on the node.
```
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.1.20    | node-to-node mesh | up    | 17:44:47 | Established |
| 10.0.1.31    | node-to-node mesh | up    | 17:44:46 | Established |
| 10.0.1.10    | global            | up    | 20:35:04 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

The output above shows that Calico is now peered with the bastion node (`10.0.1.10`). This means Calico can share routes to and learn routes from bastion node.

In a real-world on-prem deployment you would typically configure Calico nodes within a rack to peer with the ToRs (Top of Rack) routers, and the ToRs are then connected to the rest of the enterprise or data center network. In this way, if desired, pods can be reached from anywhere in then network. You could even go as far as giving some pods public IP address and have them addressable from the Internet if you wanted to.

We're done with adding the peers, so exit from worker1 to return back to bastion node.
```
exit
```

### Configure a namespace to use externally routable IP addresses

Calico supports annotations on both namespaces and pods that can be used to control which IPPool (or even which IP address) a pod will receive its address from when it is created. In this example, we're going to create a namespace to host externally routable Pods.


Let's create a namespace with the required Calico IPAM annotations. Examine the namespace configurations. Note how Calico IPAM annotation `cni.projectcalico.org/ipv4pools` is used with the name of IPPool `external-pool` to allocate IP addresses from IPPool `external-pool` to pods deployed in this namespace.

```
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    cni.projectcalico.org/ipv4pools: '["external-pool"]'
  name: external-ns
EOF

```

You should receive an output similar to the following.

```
namespace/external-ns created
```

Before deploying nginx to test the routing for pods deployed in `external-pool` IPPool from outside the cluster, let's examine the routing table of `bastion` node. You should receive an output similar to the following. At this point, `bastion` node should have no routing info for `external-pool` IPPool as there has not been any pods receiving their IP addresses from that IPPool.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
```


Now, deploy a nginx example pod in the `external-ns` namespace along with a simple network policy that allows ingress on port 80.

```
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: external-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent

---

kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: nginx
  namespace: external-ns
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
EOF
```

You should receive an output similar to the following.

```
deployment.apps/nginx created
networkpolicy.networking.k8s.io/nginx created
```
Check `bastion` node's routing table again. You should have a new routing table entry.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
```

#### 2.2.4.3. Access the nginx pod from outside the cluster

Check the IP address that was assigned to our nginx pod.
```
kubectl get pods -n external-ns -o wide
```
```
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                                      NOMINATED NODE   READINESS GATES
nginx-76dd8577bc-6jqnz   1/1     Running   0          10m   10.48.2.216   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```
The output above shows that the nginx pod has an IP address from the externally routable IP Pool.

Let's try connectivity to the nginx pod from the bastion node. Please make sure to replace the IP address of nginx pod from your lab in the following command.

```
curl 10.48.2.216
```

This should have succeeded showing that the nginx pod is directly routable on the broader network.

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

If you would like to see IP allocation stats from Calico-IPAM, run the following command.

```
calicoctl ipam show
```

There is one IP address in use from the range `10.48.2.0/24`, which is for our nginx pod.
```
+----------+--------------+-----------+------------+------------+
| GROUPING |     CIDR     | IPS TOTAL | IPS IN USE |  IPS FREE  |
+----------+--------------+-----------+------------+------------+
| IP Pool  | 10.48.0.0/24 |       256 | 9 (4%)     | 247 (96%)  |
| IP Pool  | 10.48.2.0/24 |       256 | 1 (0%)     | 255 (100%) |
+----------+--------------+-----------+------------+------------+
```

> __You have completed Lab2.2 and you should have by now a strong understanding of k8s networking__
