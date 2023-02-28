# Enabling End to End Encryption with WireGuard

## Installing WireGuard

Before enabling end-to-end encryption with Calico, you must first install WireGuard. Please refer to the instructions available here for your OS: https://www.wireguard.com/install/

SSH into the each of the 3 nodes (control1, worker1, worker2) and run the following command:
```bash
sudo apt install wireguard
```

Before enabling the encryption feature, test to ensure that the wireguard module is loaded on each of the 3 nodes (control1, worker1, worker2) using the following command:

```bash
sudo lsmod | grep wireguard
```

The output should look something like this:

```bash
ubuntu@ip-10-0-1-30:~$ sudo lsmod | grep wireguard
wireguard              81920  0
curve25519_x86_64      49152  1 wireguard
libchacha20poly1305    16384  1 wireguard
libblake2s             16384  1 wireguard
ip6_udp_tunnel         16384  1 wireguard
udp_tunnel             20480  1 wireguard
libcurve25519_generic    49152  2 curve25519_x86_64,wireguard
```

Now that we've enabled the wireguard module on the 3 nodes we can enable encrpytion through the 'felixconfiguration' in Calico.

## Enable End to End Encryption

To enable end-to-end encryption, we will patch the 'felixconfiguration' with the 'wireguardEnabled' option.

```bash
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```

To validate, we will need to check the node status for Wireguard entries with the following command:

```bash
kubectl get nodes -o yaml | grep 'kubernetes.io/hostname\|Wireguard'
```

Which will give us the following output showing the nodes hostname as well as the Wireguard Interface Address and PublicKey:

```bash
tigera@bastion:~$ kubectl get nodes -o yaml | grep 'kubernetes.io/hostname\|Wireguard'
      projectcalico.org/IPv4WireguardInterfaceAddr: 10.48.115.87
      projectcalico.org/WireguardPublicKey: gsxCZHLjKBjJozxRFiKjbMB3QtVTluQDmbvQVfANmXE=
      kubernetes.io/hostname: ip-10-0-1-20.ca-central-1.compute.internal
            f:kubernetes.io/hostname: {}
            f:projectcalico.org/IPv4WireguardInterfaceAddr: {}
            f:projectcalico.org/WireguardPublicKey: {}
      projectcalico.org/IPv4WireguardInterfaceAddr: 10.48.127.254
      projectcalico.org/WireguardPublicKey: HmNsTyzg7TKvs/Fh0AmA0VEgtS+Ij6xBHqvzXO5VfmA=
      kubernetes.io/hostname: ip-10-0-1-30.ca-central-1.compute.internal
            f:kubernetes.io/hostname: {}
            f:projectcalico.org/IPv4WireguardInterfaceAddr: {}
            f:projectcalico.org/WireguardPublicKey: {}
      projectcalico.org/IPv4WireguardInterfaceAddr: 10.48.116.146
      projectcalico.org/WireguardPublicKey: lDSws3G/G1KP76BGGRpVSXBnTt5N6FCqOodzTUUWs0I=
      kubernetes.io/hostname: ip-10-0-1-31.ca-central-1.compute.internal
            f:kubernetes.io/hostname: {}
            f:projectcalico.org/IPv4WireguardInterfaceAddr: {}
            f:projectcalico.org/WireguardPublicKey: {}
```

On each node we can also view the new interface created by Wireguard with the 'wg' command:

```bash
ubuntu@ip-10-0-1-30:~$ sudo wg
interface: wireguard.cali
  public key: HmNsTyzg7TKvs/Fh0AmA0VEgtS+Ij6xBHqvzXO5VfmA=
  private key: (hidden)
  listening port: 51820
  fwmark: 0x200000

peer: gsxCZHLjKBjJozxRFiKjbMB3QtVTluQDmbvQVfANmXE=
  endpoint: 10.0.1.20:51820
  allowed ips: 10.48.115.64/26, 10.48.115.87/32, 10.0.1.20/32
  latest handshake: 1 minute, 26 seconds ago
  transfer: 22.09 MiB received, 12.46 MiB sent

peer: lDSws3G/G1KP76BGGRpVSXBnTt5N6FCqOodzTUUWs0I=
  endpoint: 10.0.1.31:51820
  allowed ips: 10.48.116.128/26, 10.48.116.146/32, 10.0.1.31/32
  latest handshake: 1 minute, 27 seconds ago
  transfer: 23.64 MiB received, 13.21 MiB sent
```

## See WireGuard in Action

For verifying the encryption we will spin up two pods (pod-1 in worker1 and pod-2 in worker2), initiate the traffic between the two pods and view the traffic by using the tcpdump command.

Create pod-1 and pod-2 on respective worker nodes:
```bash
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-1
  name: pod-1
spec:
  containers:
  - image: praqma/network-multitool
    name: pod-1
  nodeName: ip-10-0-1-30.ca-central-1.compute.internal
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-2
  name: pod-2
spec:
  containers:
  - image: praqma/network-multitool
    name: pod-2
  nodeName: ip-10-0-1-31.ca-central-1.compute.internal
---
EOF
```
Validate that the pods got scheduled in the correct nodes:
```bash
kubectl get pods -o wide
```
```bash
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE                                         NOMINATED NODE   READINESS GATES
pod-1   1/1     Running   0          9s    10.48.0.101   ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
pod-2   1/1     Running   0          9s    10.48.0.217   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```
Open two new tabs in the browser (`http://<LABNAME>.lynx.tigera.ca`) and use the bastion host to SSH into worker1 and worker2 nodes.
Run the following commands on **both** worker1 and worker2 to see the flow of traffic on wiregaurd interface tunnel:
```bash
sudo tcpdump -i wireguard.cali host 10.48.0.101
```
On the bastion host, exec into the pod-1 and ping the pod-2's IP address (10.48.0.217 - This IP might be different in your cluster).
```bash
kubectl exec -it pod-1 -- bash
```
```bash
bash-5.1# ping -c5 10.48.0.217
PING 10.48.0.217 (10.48.0.217) 56(84) bytes of data.
64 bytes from 10.48.0.217: icmp_seq=1 ttl=62 time=0.930 ms
64 bytes from 10.48.0.217: icmp_seq=2 ttl=62 time=0.605 ms
64 bytes from 10.48.0.217: icmp_seq=3 ttl=62 time=0.502 ms
64 bytes from 10.48.0.217: icmp_seq=4 ttl=62 time=0.635 ms
64 bytes from 10.48.0.217: icmp_seq=5 ttl=62 time=0.536 ms

--- 10.48.0.217 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4079ms
rtt min/avg/max/mdev = 0.502/0.641/0.930/0.151 ms
```
On the worker1 and worker2 you should see the packet captures on wireguard.cali interface which indicates the traffic is being encrypted on the wire.
```bash
ubuntu@ip-10-0-1-30:~$ sudo tcpdump -i wireguard.cali host 10.48.0.101
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wireguard.cali, link-type RAW (Raw IP), capture size 262144 bytes
06:34:06.145612 IP ip-10-48-0-101.ca-central-1.compute.internal > ip-10-48-0-217.ca-central-1.compute.internal: ICMP echo request, id 26, seq 1, length 64
06:34:06.148131 IP ip-10-48-0-217.ca-central-1.compute.internal > ip-10-48-0-101.ca-central-1.compute.internal: ICMP echo reply, id 26, seq 1, length 64
06:34:07.147298 IP ip-10-48-0-101.ca-central-1.compute.internal > ip-10-48-0-217.ca-central-1.compute.internal: ICMP echo request, id 26, seq 2, length 64
06:34:07.147915 IP ip-10-48-0-217.ca-central-1.compute.internal > ip-10-48-0-101.ca-central-1.compute.internal: ICMP echo reply, id 26, seq 2, length 64
06:34:08.163928 IP ip-10-48-0-101.ca-central-1.compute.internal > ip-10-48-0-217.ca-central-1.compute.internal: ICMP echo request, id 26, seq 3, length 64
06:34:08.164407 IP ip-10-48-0-217.ca-central-1.compute.internal > ip-10-48-0-101.ca-central-1.compute.internal: ICMP echo reply, id 26, seq 3, length 64
06:34:09.187823 IP ip-10-48-0-101.ca-central-1.compute.internal > ip-10-48-0-217.ca-central-1.compute.internal: ICMP echo request, id 26, seq 4, length 64
06:34:09.188201 IP ip-10-48-0-217.ca-central-1.compute.internal > ip-10-48-0-101.ca-central-1.compute.internal: ICMP echo reply, id 26, seq 4, length 64
06:34:10.211824 IP ip-10-48-0-101.ca-central-1.compute.internal > ip-10-48-0-217.ca-central-1.compute.internal: ICMP echo request, id 26, seq 5, length 64
06:34:10.212667 IP ip-10-48-0-217.ca-central-1.compute.internal > ip-10-48-0-101.ca-central-1.compute.internal: ICMP echo reply, id 26, seq 5, length 64
```
[Encrypt Data in Transit](https://docs.tigera.io/compliance/encrypt-cluster-pod-traffic)

[Install Wireguard](https://www.wireguard.com/install/)
