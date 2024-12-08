# The Kubernetes Control Plane Installation v1.30.3 on Oracle Linux

0. Run commands as root
```shell
sudo -i
```

1. Update packages
```shell
yum -y update
```

2. Disable the firewall
```shell
systemctl stop firewalld && \
systemctl disable firewalld && \
systemctl status firewalld
```

3. Disable swap
```shell
swapoff -a
```

4. Ensure that swap is absent in /etc/fstab
```shell
cat /etc/fstab  | grep swa
```

5. [Enable routing](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional)
```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sysctl --system

sysctl net.ipv4.ip_forward
```

6. Install a container runtime
```shell
yum install -y yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

yum install containerd.io -y

systemctl status containerd
```

7. Check the sandbox version and configure as necessary
```shell
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sed 's/sandbox_image = "registry.k8s.io\/pause:3.8"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml

grep sandbox_image /etc/containerd/config.toml && grep SystemdCgroup /etc/containerd/config.toml
```

8. Start and enable containerd
```shell
systemctl start containerd && \
systemctl enable containerd && \
systemctl status containerd
```

9. [Install](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) kubeadm, kubectl, and kubelet
```shell
# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
grep permissive /etc/selinux/config

# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

10. Start and enable kubelet
```shell
systemctl start kubelet.service && \
systemctl enable kubelet.service && \
systemctl status kubelet.service
```

11. Choose a CIDR for pods, e.g., 10.130.0.0/19.

12. Prepare* a configuration for kubeadm
```shell
cat << EOF > kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.30.3
controlPlaneEndpoint: my-api-endpoint.example.com:6443
networking:
  podSubnet: 10.130.0.0/19
apiServer:
  certSANs:
  - "127.0.0.1"
  - "localhost"
  - "10.10.10.10"
  - "10.10.10.11"
  - "10.10.10.12"
  - "10.10.10.13"
EOF
```

- `10.10.10.10` - a Virtual IP address created by Keepalived
- `10.10.10.11`, `10.10.10.12`, `10.10.10.13` - IP addresses of control plane nodes
- `my-api-endpoint.example.com` - the domain for your Kube API server
- `my-api-endpoint.example.com` resolves to `10.10.10.10`

13. Verify that /var/lib/etcd is empty. If it isn't, delete all contents.

14. Run the installation on the first control plane host only.
```shell
kubeadm init --config kubeadm-config.yaml --upload-certs
```

15. Save the provided information:
```
Your Kubernetes control-plane has initialized successfully!
.....
```

16. Set up access to the Kubernetes cluster by configuring the kubeconfig file.
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```