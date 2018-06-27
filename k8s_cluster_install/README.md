# Usefull commands, on master, can be used after master installed to inspect the cluster

Get all system pods - 
```
kubectl --namespace=kube-system get pods
```
Get logs of a specifig pod - 
```
kubectl --namespace=kube-system logs <pod id>
```
Get nodes - 
```
kubectl get nodes
```
Get deployments - 
```
kubectl get deploy
```
Delete node - 
```
kubectl delete node <node host name>
```
Delete deployment - 
```
kubectl delete deploy <deployment name>
```
 
## Do the below on all nodes

### set the host name of the master (and for all the other nodes)
```
hostnamectl set-hostname kubemaster
```

### edit /etc/hosts on all nodes to have the hostnames of one another
```
vim /etc/hosts
```

### disable SELinux
```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```
### disable swap
```
swapoff -a
```

### edit /etc/fstab and remark the swap parition
```
vim /etc/fstab
```

### load the bridge kernel module
```
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### install needed packages
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce
yum install -y kubelet kubeadm kubectl
```

### add cgroups
```
sed -i 's/cgroup-driver=systemd/cgroup-driver=cgroupfs/g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet
systemctl enable kubelet
systemctl enable docker
reboot
```

### open the needed ports on firewall,make sure firewalld service is enabled and working
```
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --reload
```

## On the master node
```
kubeadm init --apiserver-advertise-address=<master public up> --pod-network-cidr=10.11.1.0/24 --service-cidr=10.12.1.0/24 --ignore-preflight-errors=cri
```
### after the above is done, keep the join output command, you will need it later to join the nodes
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### install flannel
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### install UI dashboard
```
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

### create cert for UI login
```
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```
#### copy the kubecfg.p12 and insert to browser

### create a user
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF
```

### set auth
```
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```

### get the token to login to the UI
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

### UI URL - 
```
https://<master public ip>:<api server port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```
## On the cluster nodes
### join the node to the cluster , token and cert are presented after the kubeadm init command
```
kubeadm join <master public ip>:6443 --token <token> --discovery-token-ca-cert-hash <cert> 
```

## On the master , check if the flannel pods are running OK, if not, probably needs to set their CIDR, according to what was provieded in the kubeadm init command
```
kubectl patch node <node host name> -p '{"spec":{"podCIDR":"<pods cidr>"}}'
```

![alt text](https://github.com/yanivtal5/k8s/blob/master/k8s_cluster_install/kube-dashboard.png)
