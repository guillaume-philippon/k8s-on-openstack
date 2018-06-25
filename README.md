# k8s-on-openstack

*Note: Starting VM and retrieve information about can be done on keystone portal instead of CLI interface*

## Check your openstack environnement
```bash
box:~ user$ openstack keypair list
+-----------------------------+-------------------------------------------------+
| Name                        | Fingerprint                                     |
+-----------------------------+-------------------------------------------------+
| <your-ssh-key-on-openstack> | aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99 |
+-----------------------------+-------------------------------------------------+
```

## Start 5 VMs
```bash
box:~ user$ openstack server create --image CentOS-7-x86_64-GenericCloud-1802 --min 5 --max 5 --network public --flavor os.2 --key-name <your-ssh-key-on-openstack> k8s-cluster
```

## Retrieve VMs IP
```bash
box:~ user$ openstack server list
+--------------------------------------+---------------+--------+-----------------------+-----------------------------------+--------+
| ID                                   | Name          | Status | Networks              | Image                             | Flavor |
+--------------------------------------+---------------+--------+-----------------------+-----------------------------------+--------+
| aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee | k8s-cluster-5 | ACTIVE | public=192.168.10.10  | CentOS-7-x86_64-GenericCloud-1802 | os.2   |
| aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee | k8s-cluster-4 | ACTIVE | public=192.168.10.11  | CentOS-7-x86_64-GenericCloud-1802 | os.2   |
| aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee | k8s-cluster-3 | ACTIVE | public=192.168.10.12  | CentOS-7-x86_64-GenericCloud-1802 | os.2   |
| aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee | k8s-cluster-2 | ACTIVE | public=192.168.10.13  | CentOS-7-x86_64-GenericCloud-1802 | os.2   |
| aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee | k8s-cluster-1 | ACTIVE | public=192.168.10.14  | CentOS-7-x86_64-GenericCloud-1802 | os.2   |
+--------------------------------------+---------------+--------+-----------------------+-----------------------------------+--------+
```

## Define role for your VM
  * k8s-cluster-1 (192.168.10.14) : NFS server + haproxy
  * k8s-cluster-2 (192.168.10.13): kubernetes master
  * k8s-cluster-3 (192.168.10.12): kubernetes-node
  * k8s-cluster-4 (192.168.10.11): kubernetes-node
  * k8s-cluster-5 (192.168.10.10): kubernetes-node

## Connect and update each VMs
On every VM, connected throught ssh and run
```bash
box:~ user$ ssh centos@<my-ip>
[centos@k8s-cluster-<id> ~]$ sudo -i
[root@k8s-cluster-<id> ~]# yum -y update
[root@k8s-cluster-<id> ~]# yum -y update
[...]
[root@k8s-cluster-<id> ~]# reboot
```

You also need a hosts file looks like
```bash
[root@k8s-cluster-<id> ~]# cat /etc/hosts 
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.10.14 k8s-cluster-1
192.168.10.13 k8s-cluster-2
192.168.10.12 k8s-cluster-3
192.168.10.11 k8s-cluster-4
192.168.10.10 k8s-cluster-5
```

## NFS infrastructure
## Configure NFS server
*Note: see https://www.server-world.info/en/note?os=CentOS_7&p=nfs*
```bash
[root@k8s-cluster-1 ~]# echo "/mnt *(rw,no_root_squash)" > /etc/exports
[root@k8s-cluster-1 ~]# systemctl enable rpcbind nfs-server
[root@k8s-cluster-1 ~]# systemctl restart rpcbind nfs-server
[root@k8s-cluster-1 ~]# mkdir /mnt/binderhub
[root@k8s-cluster-1 ~]# chmod gou+rwx /mnt/binderhub
```
**Warning: You should be careful with /etc/exports syntax. You should NOT have space between options**

## Test NFS server
```bash
[root@k8s-cluster-2 ~]# mount -t nfs 192.168.10.14:/mnt /mnt
[root@k8s-cluster-2 ~]# df |grep mnt
192.168.10.14:/mnt    20960256 1259520   19700736   7% /mnt
[root@k8s-cluster-2 ~]# umount /mnt
```

## Configure kubernetes
### Installing docker (k8s-cluster-2 to k8s-cluster-5)
*Note: see https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce*
```bash
[root@k8s-cluster-<id 2+> ~]#  yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
[...]
[root@k8s-cluster-<id 2+> ~]#  yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
[...]
[root@k8s-cluster-<id 2+> ~]#  yum install -y docker-ce-17.12.0.ce-1.el7.centos
[root@k8s-cluster-<id 2+> ~]#  systemctl enable docker
[root@k8s-cluster-<id 2+> ~]#  systemctl start docker
[root@k8s-cluster-<id 2+> ~]#  # Test docker installation
[root@k8s-cluster-<id 2+> ~]#  docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
9bb5a5d4561a: Pull complete
Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```
### Configure kubernetes
*Note: see https://kubernetes.io/docs/tasks/tools/install-kubeadm/*
#### Common
```bash
[root@k8s-cluster-<id 2+> ~]#  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
[root@k8s-cluster-<id 2+> ~]#  setenforce 0
[root@k8s-cluster-<id 2+> ~]#  yum install -y kubelet kubeadm kubectl
[root@k8s-cluster-<id 2+> ~]#  sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[root@k8s-cluster-<id 2+> ~]#  systemctl enable kubelet && systemctl start kubelet
[root@k8s-cluster-<id 2+> ~]#  cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
[root@k8s-cluster-<id 2+> ~]#  sysctl --system
```

#### Master
*Note: see https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/*
```bash
[root@k8s-cluster-2 ~]# kubeadm init --pod-network-cidr=10.244.0.0/16
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.10.11:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha>
[root@k8s-cluster-2 ~]# mkdir -p $HOME/.kube
[root@k8s-cluster-2 ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-cluster-2 ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
[root@k8s-cluster-2 ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
[root@k8s-cluster-2 ~]# kubectl get node
NAME                         STATUS     ROLES     AGE       VERSION
k8s-cluster-2.lal.in2p3.fr   Ready      master    18m       v1.10.5
```
**Note: The last command is important, you will use it to join node to master**

#### Node
```bash
[root@k8s-cluster-<id 3+> ~]# kubeadm join 192.168.10.11:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha>
```

#### Verify kubernetes installation (kubernetes master)
*Note: Every command will now be launch on kubernetes master. There is no need to connect to k8s nodes*
```bash
[root@k8s-cluster-2 ~]# kubectl get node
NAME                         STATUS    ROLES     AGE       VERSION
k8s-cluster-2.lal.in2p3.fr   Ready     master    24m       v1.10.5
k8s-cluster-3.lal.in2p3.fr   Ready     <none>    7m        v1.10.5
k8s-cluster-4.lal.in2p3.fr   Ready     <none>    5m        v1.10.5
k8s-cluster-5.lal.in2p3.fr   Ready     <none>    7m        v1.10.5
```

## Configure binderhub
*Note: see https://binderhub.readthedocs.io/en/latest/*
*Note: every command will be launch on k8s master*

### Kubernetes initialisation
```bash
[root@k8s-cluster-2 ~]# export PATH=$PATH:/usr/local/bin # You can add it in .bashrc
[root@k8s-cluster-2 ~]# curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
[root@k8s-cluster-2 ~]# kubectl --namespace kube-system create serviceaccount tiller
[root@k8s-cluster-2 ~]# kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
[root@k8s-cluster-2 ~]# helm init --service-account tiller


[root@k8s-cluster-2 ~]# # Test helm installation
[root@k8s-cluster-2 ~]# helm version
[root@k8s-cluster-2 ~]# kubectl --namespace=kube-system patch deployment tiller-deploy --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
```
### Binderhub configuration files
```bash
[root@k8s-cluster-2 ~]# mkdir binderhub
[root@k8s-cluster-2 ~]# cd binderhub
[root@k8s-cluster-2 ~]# openssl rand -hex 32 > apiToken.txt
[root@k8s-cluster-2 ~]# openssl rand -hex 32 > secretToken.txt
```
#### Secret.yaml
```yaml
jupyterhub:
    hub:
      services:
        binder:
          apiToken: "<content of apiToken.txt>"
    proxy:
      secretToken: "<content of secretToken.txt>"
hub:
  services:
    binder:
      apiToken: "<content of apiToken.txt>"
registry:
  username: dockerhub-username
  password: dockerhub-password
```

#### Config.yaml
*Note: enabling registry break binderhub. To allow binderhub you need to disable registry*
```yaml
registry:
  enabled: true
  prefix: registry.hub.docker.com/sinese
  host: https://registry.hub.docker.com
  authHost: https://index.docker.io/v1
  authTokenUrl: https://auth.docker.io/token?service=registry.docker.io
hub:
  url: http://192.168.10.14:8080
repo2dockerImage: jupyter/repo2docker:2dc4874
baseUrl: /
jupyterhub:
  hub:
    proxy:
      service:
        type: NodePort
binderhub:
  service:
    type: NodePort
```

#### Load binderhub
```bash
[root@k8s-cluster-2 ~]# helm repo add jupyterhub https://jupyterhub.github.io/helm-chart
[root@k8s-cluster-2 ~]# helm repo update
[root@k8s-cluster-2 ~]# helm install jupyterhub/binderhub --version=v0.1.0-856e3e6  --name=binder  -f secret.yaml -f config.yaml
```
Change LoadBalancer to NodePort for service/binder and service/proxy
*Note: Should be done by config.yaml*
```bash
[root@k8s-cluster-2 ~]# kubectl edit service/binder
# Change type: LoadBalancer to NodePort
[root@k8s-cluster-2 ~]# kubectl edit service/binder
# Change type: LoadBalancer to NodePort
```

We now need to create a persistent volume claim for hub pod. Create a file hub-db-dir.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: binder
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 1Gi
  nfs:
    server: 192.168.10.14
    path: "/mnt/binderhub"
```
And now launch it
```bash
[root@k8s-cluster-2 ~]# kubectl apply -f hub-db-dir.yaml
```

To modify binderhub configuration you need to user helm
```bash
[root@k8s-cluster-2 ~]# helm upgrade binder jupyterhub/binderhub --version=v0.1.0-856e3e6 -f secret.yaml -f config.yaml
```

## Configure haproxy
We now need to configure haproxy to access to binderhub. On k8s master note proxy-public and binder port used
```bash
[root@k8s-cluster-2 ~]# kubectl get service | grep 'binder\|public''
binder         NodePort    10.100.212.148   <none>        80:30896/TCP   12m
proxy-public   NodePort    10.109.153.126   <none>        80:31058/TCP   12m
```
binder port is bound to **30896** and proxy is bound to **31058**

on k8s-cluster-1 install haproxy
```bash
[root@k8s-cluster-1 ~]# yum install -y haproxy
```

on k8s-cluster-1 configure /etc/haproxy/haproxy.conf
```ini
# Simple configuration for an HTTP proxy listening on port 80 on all
    # interfaces and forwarding requests to a single backend "servers" with a
    # single server "server1" listening on 127.0.0.1:8000
    global
        daemon
        maxconn 256

    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms

    listen http-in
        bind *:80
        server server1 192.168.10.10:30896 maxconn 32

   listen http-in
        bind *:8080
        server server1 192.168.10.10:31058 maxconn 32
```
*Note: To have a HAproxy, you need to add all node IP. With this configuration, all traffic will be sent to k8s-cluster-5, but all k8s node is a valid service*

and restart haproxy
```bash
[root@k8s-cluster-1 ~]# systemctl enable haproxy
[root@k8s-cluster-1 ~]# systemctl restart haproxy
```
