# The Kubernetes Worker Node installation

1. Run commands as root
```shell
sudo -i

```

2. Update packages
```shell
yum -y update
```

3. Execute the same commands as on the control plane node:
- Disable the firewall
- Disable swap
- Verify that swap is absent in /etc/fstab.
- Enable routing
- Install a container runtime
- Check the sandbox version and configure as necessary.
- Start and enable containerd
- Install kubeadm, kubectl, and kubelet
- Start and enable kubelet

4. Add the worker node to the cluster
```shell
sudo -i
kubadm join ...
```

To generate the `kubeadm join` command, run:
```shell
kubeadm token create --print-join-command
```
