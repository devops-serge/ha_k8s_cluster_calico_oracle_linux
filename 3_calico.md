# Calico Installation with BGP
1. Install the Tigera Calico operator and custom resource definitions.
```shell
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
```

2. Download the custom resources.
```shell
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml
```

3. Set your CIDR and apply it to the cluster. For more information: [https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
```
cat << EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    bgp: Enabled
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 24
      cidr: 10.130.0.0/19
      encapsulation: None
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

`10.130.0.0/19` - set your CIDR.

4. Configure BGP
```
cat << EOF | kubectl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  asNumber: 65535
  bindMode: NodeIP
```

5. Install [calicoctl](https://docs.tigera.io/calico/latest/operations/calicoctl/install#install-calicoctl-as-a-binary-on-a-single-host)


6. Verify
- the current IP address management (IPAM) configuration in Calico
```shell
calicoctl ipam show --show-blocks
```

- Node status
```shell
calicoctl node status
```

- BGP configuration
```shell
kubectl describe bgpconfiguration
```