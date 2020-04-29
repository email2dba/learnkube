 
Install Kubenetes on centos 7 using kubeadm on virutalBox
==========================================================

Reference
---------
#I have followed these links to install debug the issues
#and if you face any issues during
# installation these links may help to fix. 
#
#https://www.linuxtechi.com/install-kubernetes-1-7-centos7-rhel7/
#https://phoenixnap.com/kb/how-to-install-kubernetes-on-centos
#https://github.com/kelseyhightower/kubernetes-the-hard-way
#https://github.com/weaveworks/weave/issues/3363
#
#
Notes
------
#Systems used to install kubernetes 

ServerType  name           OS     IPaddress     
Phsyical    ilakkasrvr01   Centos 192.168.1.75 
VirtualBox  master001      Centos 192.168.5.11
VirtualBox  worker001      Centos 192.168.5.22
VirtualBox  worker001      Centos 192.168.5.24

#VirtualBox 6.1 insalled in ilakkasrvr01 server
 
Step 0: Create virutal centos box
---------------------------------
#in ilakkasrvr01 server login as non-root user
# (user name : enduser) and create first Virtaul node 

mkdir -p /var/loginuser/kubernetes/centos 
cd /var/loginuser/kubernetes/centos 

vagrant init centos/7
vagrant up

Step 1:Login into Guest OS 
------------------------------
vagrant global-status 

vagrant ssh  default

<<Change root password and keep it safe>>
sudo su - 
passwd root 

hostnamectl set-hostname master001
exec bash
#
{
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
}

#reboot  master001
init 6

Step 2 :setup networking using this Document:
--------------------------------------------
network_virtualboxsetup001.pdf

#login into master001 
#verify  networking is working
ping -c2 www.google.com

Step 3: setup firewall rules
----------------------------
#login as root user in master001 
# run update  command to update all rpm 
{
yum update 
}

{
 systemctl start  firewalld
 systemctl enable firewalld
 firewall-cmd --permanent --add-port=6443/tcp
 firewall-cmd --permanent --add-port=2379-2380/tcp
 firewall-cmd --permanent --add-port=10250/tcp
 firewall-cmd --permanent --add-port=10251/tcp
 firewall-cmd --permanent --add-port=10252/tcp
 firewall-cmd --permanent --add-port=10255/tcp

 firewall-cmd --reload
}

{
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
}

Step4 : Update /etc/hosts
-------------------------
#login as root user in master001 
[root@master001 ~]# cat /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.5.11  master001 
192.168.5.22  worker001
192.168.5.24  worker002
192.168.1.75  ilakkasrvr01  <<This is my Host server>>


Step 5: Configure Kubernetes Repository
---------------------------------------
#login as root user in master001 
#Kubernetes packages are not available in the default CentOS 7 & RHEL 7 repositories,
#Use below command to configure its package repositories.

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF


Step 6: Clone master001 as worker001 and worker002
--------------------------------------------------
#shutdown  master001
init 0

#clone master001 as worker001 and worker002
Note: I have used VirtualBox GUI to clone and skipping these steps.

Step 7: Install Kubeadm and Docker in master001
-----------------------------------------------
#Bring the master001 UP using GUI or this command.
VBoxManage list vms
VBoxManage startvm master001 --type headless

#since we have  configured the package repositories, run the following
#command to install kubeadm and docker packages.

yum install kubeadm docker -y
#Start and enable kubectl and docker service
systemctl restart docker  && systemctl enable docker
systemctl restart kubelet && systemctl enable kubelet

Step 8: Initialize Kubernetes Master with 'kubeadm init'
--------------------------------------------------------
#Run the beneath command to initialize and setup kubernetes master.
# I have used root user account for all and may not need to be root account to configure kubenetes.
kubeadm init --apiserver-advertise-address=192.168.5.11 --pod-network-cidr=10.96.0.0/12

#Following information you will receive and plese keep it safe.

#Then you can join any number of worker nodes by running the following on each as root:
#kubeadm join 192.168.5.11:6443 --token gmlu6z.virmyt5pe8nnf6fj \
#    --discovery-token-ca-cert-hash sha256:0f3059df758559b31077101969f8c77e40d9510ad8705e546aa8066700b0fd83 

#NOTE: this tocker is valid only for 24 hours you must finish all steps before 24 hours otherwise
#create new token in master001 server using "kubeadm token create ; kubeadm token list "

#I copied admin.conf to $HOME/.kube/config
cat /etc/kubernetes/admin.conf 
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

Step 9: Deploy pod network to the cluster
------------------------------------------
#in master001 node 
export kubever=$(kubectl version | base64 | tr -d '\n')
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=${kubever}\&env.IPALLOC_RANGE=10.96.0.0/12"  

serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created

#verify the containers status
kubectl get pods  -n kube-system
NAME                                READY   STATUS    RESTARTS   AGE    IP             NODE        NOMINATED NODE   READINESS GATES
coredns-66bff467f8-92jf5            1/1     Running   0          65m    10.96.0.2      master001   <none>           <none>
coredns-66bff467f8-mxkgr            1/1     Running   0          65m    10.96.0.3      master001   <none>           <none>
etcd-master001                      1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-apiserver-master001            1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-controller-manager-master001   1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-proxy-d8bgm                    1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-scheduler-master001            1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
weave-net-vvwst                     2/2     Running   0          60m    192.168.5.11   master001   <none>           <none>


Step 10. Set IP address on Worker nodes
---------------------------------------
# Shutdown master001. Because, we have cloned from master and worker001 and worker002
# currently having hostname master001 name and its ip addresses.
#in  master001 as root user
init 0

# startup worker001 from local server
VBoxManage startvm worker001 --type headless

#change hostname to worker001  << currentl master001 >>
hostnamectl set-hostname  worker001
#change  ip address to 192.168.5.22
vi /etc/sysconfig/network-scripts/ifcfg-eth1 
#change from 192.168.5.11 into 192.168.5.22

# startup worker002 from local server
VBoxManage startvm worker002 --type headless

#change hostname to worker002   << currentl master001 >>
hostnamectl set-hostname  worker002
#change  ip address to 192.168.5.24

vi /etc/sysconfig/network-scripts/ifcfg-eth1 
#change from 192.168.5.11 into 192.168.5.24

Step 11]start Docker on worker nodes 
------------------------------------
# since cloned from master001 before execute "kubectl init" on master001,
# we no need to configure to open port again in firewalld ,
# kubernetes repositories  and kubeadm and docker installation
# in worker001 and worker002
yum install kubeadm docker -y
systemctl restart docker && systemctl enable docker
systemctl restart kubelet && systemctl enable kubelet

Step 12: Now Join worker nodes to master node 
----------------------------------------------
#To join worker nodes to Master node, a token is required. 
#Whenever kubernetes master initialized, 
#      then in the output we get command and token.
#Copy that command and run on both nodes.
#in worker001 and worker002
kubeadm join 192.168.5.11:6443 --token gmlu6z.virmyt5pe8nnf6fj \
    --discovery-token-ca-cert-hash sha256:0f3059df758559b31077101969f8c77e40d9510ad8705e546aa8066700b0fd83 


Step 13: Verify information in master001
---------------------------------------------
# in master001
[root@master001 ~]# ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
10.96.0.0/12 dev weave proto kernel scope link src 10.96.0.1 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.5.0/24 dev eth1 proto kernel scope link src 192.168.5.11 metric 101 

[root@master001 ~]# kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
master001   Ready    master   62m   v1.18.2
worker001   Ready    <none>   53m   v1.18.2
worker002   Ready    <none>   71s   v1.18.2


kubectl get pods  -n kube-system
NAME                                READY   STATUS    RESTARTS   AGE    IP             NODE        NOMINATED NODE   READINESS GATES
coredns-66bff467f8-92jf5            1/1     Running   0          65m    10.96.0.2      master001   <none>           <none>
coredns-66bff467f8-mxkgr            1/1     Running   0          65m    10.96.0.3      master001   <none>           <none>
etcd-master001                      1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-apiserver-master001            1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-controller-manager-master001   1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-proxy-5wqpb                    1/1     Running   0          5m4s   192.168.5.24   worker002   <none>           <none>
kube-proxy-d8bgm                    1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
kube-proxy-gxpj6                    1/1     Running   1          57m    192.168.5.22   worker001   <none>           <none>
kube-scheduler-master001            1/1     Running   0          65m    192.168.5.11   master001   <none>           <none>
weave-net-5zqlv                     2/2     Running   0          33m    192.168.5.22   worker001   <none>           <none>
weave-net-vvwst                     2/2     Running   0          60m    192.168.5.11   master001   <none>           <none>
weave-net-wptk4                     2/2     Running   1          5m4s   192.168.5.24   worker002   <none>           <none>

[root@master001 ~]# kubectl get cm --all-namespaces
NAMESPACE     NAME                                 DATA   AGE
kube-public   cluster-info                         3      174m
kube-system   coredns                              1      174m
kube-system   extension-apiserver-authentication   6      174m
kube-system   kube-proxy                           2      174m
kube-system   kubeadm-config                       2      174m
kube-system   kubelet-config-1.18                  1      174m
kube-system   weave-net                            0      141m


Step 14: Verify information in worker nodes
-------------------------------------------
# in worker001 and worker002
[root@worker001 ~]# ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
10.96.0.0/12 dev weave proto kernel scope link src 10.96.0.1 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.5.0/24 dev eth1 proto kernel scope link src 192.168.5.22 metric 101 
[root@worker001 ~]# 

[root@worker002 ~]# ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.5.0/24 dev eth1 proto kernel scope link src 192.168.5.24 metric 101 
[root@worker002 ~]# 

#--------------------------------------------------------------------------------------
#--------------------------------------------------------------------------------------

commands for Reference 
-----------------------
kubectl edit cm cluster-info  -n kube-public
kubectl describe node master-1
kubectl get events -n kube-system 
kubectl edit cm cluster-info   -n kube-public
kubectl edit cm kubeadm-config -n kube-system 
kubectl get pods  -n kube-system 
kubectl -n kube-system  get cm

ip route add 10.96.0.1/32 dev eth1 src 192.168.5.22  in worker001 
ip route add  10.96.0.1/32 dev eth1 src 192.168.5.11  in master001

kubectl get componentstatus

kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
62ea1c323571aafac5936a418ebdb1a4287efde8e5f88f36f35955366fb427ed

kubeadm token create
kubeadm token list

in worker001
kubeadm reset


##Open kube-apiserver.yaml and update server address
cat /etc/kubernetes/manifests/kube-apiserver.yaml

kubectl get nodes
kubectl describe daemonset kube-proxy -n kube-system
kubectl get  daemonset weave-net -n kube-system -oyaml

firewall-cmd --permanent --add-port=6784/tcp
firewall-cmd --reload

systemctl status kubelet

kubectl describe node worker001  

systemctl status kubelet

journalctl -xeu kubelet
sysctl -w net.bridge.bridge-nf-call-iptables=1
##Have you try to pass bridged IPv4 traffic to iptables by running sysctl -w net.bridge.bridge-nf-call-iptables=1
##sysctl net.ipv4.conf.all.forwarding=1
##echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf

systemctl restart containerd kubelet kube-proxy
 
ip route add 10.96.0.1/32 dev enp0s3
ip route add 10.96.0.1/32 dev eth1

#in local server 
#vi /etc/sysconfig/network-scripts/route-eno1
#10.96.0.1/32 via 192.168.1.75 dev eno1

#in master server 
#vi /etc/sysconfig/network-scripts/route-eth1
#10.96.0.1/32 via 192.168.5.22 dev eth1

#Tips
#VBoxManage list vms
# Power off VM
#VBoxManage controlvm worker002 poweroff
# Start VM in headless mode
#VBoxManage startvm master001 --type headless
#VBoxManage startvm worker001 --type headless
#VBoxManage startvm worker002 --type headless
