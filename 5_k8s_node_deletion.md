1. Run this on a node.
```shell
kubeadm reset
 
rm -rf /etc/cni/net.d/* && ls -l /etc/cni/net.d/
 
iptables -F && iptables -X && iptables -L
 
(OPTIONAL) delete $HOME/.kube/config  
```

2. Remove a node from the cluster.
```shell
kubectl delete node <NODE>
```