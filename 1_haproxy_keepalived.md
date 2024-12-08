# The installation of a Load Balancer (HAProxy) with Keepalived on Oracle Linux

1. Run commands as root
```shell
sudo -i
```

2. Update packages and install haproxy and keepalived.
```shell
yum -y update
yum install haproxy
yum install keepalived
```

3. Disable SELinux
```shell
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

4. Configure the kernel options
```shell
echo net.ipv4.ip_nonlocal_bind=1 >> /etc/sysctl.d/99-sysctl.conf
sysctl -p --load /etc/sysctl.d/99-sysctl.conf
```

5. Disable the firewall
```shell
systemctl stop firewalld && \
systemctl disable firewalld && \
systemctl status firewalld
```

6. Configure Keepalived on the first host
```shell
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
global_defs {
   router_id uMASTER
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 230
    priority 101
    advert_int 1

   virtual_ipaddress {
       10.10.10.10/32 dev ens192 label ens192
    }
}
EOF
```

7. Configure Keepalived on the second host
```shell
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
global_defs {
   router_id uBACKUP
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 230
    priority 100
    advert_int 1

   virtual_ipaddress {
       10.10.10.10/32 dev ens192 label ens192
    }
}

EOF
```

`10.10.10.10/32 dev ens192 label ens192`, `interface ens192` - set own settings

8. Configure HAProxy on both hosts in `/etc/haproxy/haproxy.cfg`
```shell
global
    daemon
    maxconn 30000
    log 127.0.0.1 local0 debug

defaults
    log global
    option tcplog
    maxconn 30000
    timeout connect 3000
    timeout client 30000
    timeout server 30000
    timeout check 5000
    log-format "%ci:%cp\ [%t]\ %ft\ %b/%s\ %ST\ %B\ %tsc\ %ac/%fc/%bc/%sc/%rc\ %sq/%bq\ %hr\ %hs"

frontend k8s_api
     mode tcp
     bind 10.10.10.10:6443
     default_backend k8s_api

backend k8s_api
    mode tcp
    balance roundrobin
    server master-1 10.10.10.11:6443 check
    server master-2 10.10.10.12:6443 check
    server master-3 10.10.10.13:6443 check

frontend k8s-ingress-http
    mode tcp
    bind 10.10.10.10:80
    default_backend k8s-ingress-http

frontend k8s-ingress-https
    mode tcp
    bind 10.10.10.10:443
    default_backend k8s-ingress-https

backend k8s-ingress-http
    mode tcp
    balance roundrobin
    server worker1 10.10.10.14:32231 check
    server worker2 10.10.10.15:32231 check


backend k8s-ingress-https
    mode tcp
    balance roundrobin
    server worker1 10.10.10.14:32232 check
    server worker2 10.10.10.15:32232 check

```