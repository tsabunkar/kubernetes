## Prequistes [Step-1,2]

- Create/Setup multiple VM (to be very specific 3 VMs) (Configuring VM -> Before you begin) {NOTE: Here we are creating 3 VMs with unqiue hostname, static IP Addr & installing all s/w like- kubectl,minikube,docker,etc but we could do easily with cloud}

- Installing Kubeadm in local machine
  Reference : [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/]
  - Download iso image of linux distro which will act as -> One Master Node and Two Worker Nodes
  - **ubuntu-20** vm's for all our nodes (i.e- worker and master nodes)
    - **ubuntu-20-master** (Master Node) ==> {4GB Ram, 2 CPU, 10GB HDD} [whoami- rick]
    - **ubuntu-20-worker1** (Worker Node) ==> {2GB Ram, 2 CPU, 10GB HDD} [whoami- morty]
    - **ubuntu-20-worker2** (Worker Node) ==> {2GB Ram, 2 CPU, 10GB HDD} [whoami- summer]
  - Make sure these VM instance have atleast -> 2 GB of RAM, 2 CPUs and **Bridge Network**
  - Login into VM
    - Check ssh service is running:
      - `\$ ifconfig` (inside servers- to know ip's)
      - ubuntu-20-master (Master Node): 192.168.0.103
      - ubuntu-20-worker1 (Worker Node-1): 192.168.0.104 (It will be changing check ifconfig)
      - ubuntu-20-worker2 (Master Node-2): 192.168.0.108
      - `\$ systemctl status ssh` (Should be - Active & Running)
        - (Check in all the vm's you have installed ssh-server)==> [Connection Refused Error- Make sure you have enable ssh client in all vm's : https://linuxize.com/post/how-to-enable-ssh-on-ubuntu-20-04/]
        - Steps:
          - \$ sudo apt install openssh-server
          - \$ sudo systemctl status ssh
    - (Alternative) From Host Machine- Use Putty to login with ipv4 address (HOST CLI :)
      - \$ sudo ssh -l rick 192.168.0.103 (check ur password from \$ pass)
      - \$ sudo ssh -l morty 192.168.0.104
      - \$ sudo ssh -l summer 192.168.0.108
      - (ERROR - WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!) [sol-> ssh-keygen -R "server hostname or ip" ==> \$ ssh-keygen -R <ip_v4_addr> (Run in Host Machine)]
    - Check Hostname:
      - \$ cat /etc/hostname
      - (Change the hostname by editing these above files-> as per desired-wish names)
      - Master Node hostname -> rick
      - Worker Node-1 hostname -> morty
      - Worker Node-2 hostname -> summer
    - Dsiable SWAP area
      - \$ sudo swapoff -a
      - \$ sudo nano /etc/fstab (Debian & Redhat based)
        - (Comment line- which is about `swap` as extension)
    - `\$ sudo reboot` (Reboot all vm's)
  - If Error from host cli to connect this VM ->
    - WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
    - Solution:
      - \$ ssh-keygen -R <ip_add_vm/nodes>
      - \$ ssh-keygen -R 192.168.0.108 (ip-add of vm which was giving warning)
  - To Assign Static IpAddress to all the vm's (since each reboot will dynamically change ipaddr of these vm)
    - Instead of Setting Static IpAddr we would rather prefer to create Host Only Network
    - Some Theory: ( If you launch the VMs now - There Ip Address will be changed bcoz- Bydefault ipAddress for these VM are dynamic i.e- Wifi-Router allocate dynamic or random ipAddr b/e a given range everytime this VM bootsup (Kubernetes communication wont work with Dynamic Ip Address) --> To solve this problem we need to assign static ip address [quite Risky], Alternate Solution-> **Create Host Only Network**, by disableing dynamic ipaddr (This network will act as medium for our nodes to interact b/w each other) )
    - Goto VirtualBox
    - (Navigation bar) File > Host Network Manager
    - (Select) Enable vboxnet0 Network
    - Now Go back to -> ubuntu-20-worker1 > right-click > Settings > Network
      - Adapter 2 [Make sure your vm are powered-off]
      - (select) Enable Network Adapter
      - Attached to: `Host-only Adapter` (select)
      - Name: vboxnet0
      - (Repeat above step for other nodes/vm)
    - Start All Nodes/VMs
    - \$ ifconfig (in terminal of all vms)
    - To Assgin Ip Address to Network Interface/Adapater =>
      - Sytnax: ifconfig <netwrok_adapter> <ip_addr>
      - \$ ifconfig enp0s3 192.168.99.10 (This ip can be random b/w 192.168.99.1/24 <== How do I know range, Check in- Host Network Manager) [BUT: This is **temporary** way to assign IpAddr]
      - Master Node = rick => 192.168.0.109
      - Worker Node-1 = morty => 192.168.0.110
      - Worker Node-2 = summer => 192.168.0.111
      - (To assign **Permanetly** this IpAddress on every boot-up)
        - \$ cat /etc/cloud/cloud.cfg.d/subiquity-disable-cloudinit-networking.cfg
        - \$ ip add show
        - \$ sudo nano /etc/netplan/00-installer-config.yaml
        - copy code snippet from : `00-installer-config.yaml`
        - (Make sure - requesting an IP address should be same subnet as gateway's subnet which `192.168.0.___` )
        - \$ sudo netplan get
        - \$ sudo netplan apply
        - \$ ip route show (or ifconfig)
        - \$ ping www.google.com (To check your ping)
        - now reboot and check, your ip show be STATIC and able to connect from host cli
        - Ref:
          - https://www.linuxtechi.com/assign-static-ip-address-ubuntu-20-04-lts/#:~:text=Assign%20Static%20IP%20Address%20on%20Ubuntu%2020.04%20LTS%20Desktop&text=Login%20to%20your%20desktop%20environment,and%20then%20choose%20wired%20settings.&text=In%20the%20next%20window%2C%20Choose,gateway%20and%20DNS%20Server%20IP.
          - https://pimylifeup.com/ubuntu-20-04-static-ip-address/
          - https://askubuntu.com/questions/1142301/setting-a-static-ip-address-not-connecting
      - systemctl restart network (Restart Network Settings)

---

## Configuring Steps - Install Docker & kubeadm, kubelet and kubectl Tools

3.  [Step-3] Installing Container Runtime -> Docker (If container runtime as docker is not found then kubernetes will consider - Container Runtime Interface)

- Installing runtime (To run containers in Pods, Kubernetes) (Ref- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)
  - Install Docker in All VM's Containers
  - ( Debain based) [Ref- https://docs.docker.com/engine/install/debian/]
    - \$ sudo apt-get update
    - \$ sudo apt-get install \
      apt-transport-https \
      ca-certificates \
      curl \
      gnupg-agent \
      software-properties-common
    - \$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
    - \$ sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/debian \
      \$(lsb_release -cs) \
      stable"
    - \$ sudo apt-get update
    - (
      After adding above Repo to your PPA, If you do apt-get update -> Get Error will updating then
      - \$ nano /etc/apt/sources.list ( Comment line wrt to docker in file )
      - \$ sudo add-apt-repository \
         "deb [arch=amd64] https://download.docker.com/linux/debian \
         jessie \
         stable"
      - To know different version of docker go inside repo -> https://download.docker.com/linux/debian ==> you can find - buster, jessie, stretch
      - Ref - https://stackoverflow.com/questions/41133455/docker-repository-does-not-have-a-release-file-on-running-apt-get-update-on-ubun
        )
    - NOTE: (Don't install latest Docker Engine rather supported docker version engine ==> https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.9.md#external-dependencies )
    - (We need to install specific version of Docker Engine - 17.03.x)
    - \$ apt-cache madison docker-ce (You must able to see the list of docker version )
    - sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
    - \$ sudo apt-get install docker-ce=17.03.3~ce-0~debian-jessie
    - \$ docker version
    - \$ docker image ls
  - (Install in Centos-8)
    - (Ref- https://docs.docker.com/engine/install/centos/)
    - \$ sudo yum install -y yum-utils
    - \$ sudo yum-config-manager \
      --add-repo \
      https://download.docker.com/linux/centos/docker-ce.repo
    - (To install specific docker version)
      - \$ yum list docker-ce --showduplicates | sort -r (To See all list of docker version)
      - sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
      - \$ sudo yum install docker-ce-17.03.3.ce-1.el7
      - (
        If error occurs
        - \$ yum install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm (Install CentOs-7 container.io essentials) [https://vexpose.blog/2020/04/02/installation-of-docker-fails-on-centos-8-with-error-package-containerd-io-1-2-10-3-2-el7-x86-64-is-excluded/]
        - \$ yum install docker-ce-selinux (or) yum install docker-ce
          )
      - \$ docker version
      - \$ docker image ls
      - (
        Remove docker from centos
        - \$ sudo yum remove docker \
           docker-common \
           container-selinux \
           docker-selinux \
           docker-engine
          )
  - (Install in Fedora-32)
    - \$ sudo dnf -y install dnf-plugins-core
    - \$ sudo dnf -y install dnf-plugins-core
    - $ sudo tee /etc/yum.repos.d/docker-ce.repo<<EOF
      [docker-ce-stable]
      name=Docker CE Stable - \$basearch
      baseurl=https://download.docker.com/linux/fedora/31/\$basearch/stable
      enabled=1
      gpgcheck=1
      gpgkey=https://download.docker.com/linux/fedora/gpg
      EOF
    - (Let me install not specific version of docker i.e- 17.0.3)
      - \$ sudo yum install docker-ce docker-ce-cli containerd.io
    - \$ docker version
    - \$ docker image ls

4. [Step-4] Installing kubeadm, kubelet and kubectl (Ref- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

- (For Debain)
  - \$ sudo apt-get update && sudo apt-get install -y apt-transport-https curl
  - \$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
  - \$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF
  - \$ sudo apt-get update
  - \$ sudo apt-get install -y kubelet kubeadm kubectl
  - \$ sudo apt-mark hold kubelet kubeadm kubectl
- (For CentOs & Fedora)
  - $ cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude=kubelet kubeadm kubectl
    EOF
  - \$ sudo setenforce 0
  - $ sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  - \$ sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
  - \$ sudo systemctl enable --now kubelet

---

5. [Step-5] Intialize Cluster (Ref- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#objectives)

- (Master Node) -> (All below commands to run in Debian/Master)
  - kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<static_ip_addr>
  - \$ kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.99.10 [Static IpAddr for Master Node -> 192.168.99.10]
  - (Run the commands shown in console)
  - $ mkdir -p $HOME/.kube
  - $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  - $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
  - (Copy this below command for worker nodes to connect with master)
    - kubeadm join 192.168.99.10:6443 --token ftbf8q.giu7pwvel44gqy9s \
      --discovery-token-ca-cert-hash sha256:57b378be50a35f1d8a95ad400aa77438965470bf9013fbf008a70b438b4975f1

6. [Step-6] Installing a Pod network add-on and Setup Master node (All below commands to run in Debian/Master)

   - Installing Pod Network (Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
     - (select) Calico
     - \$ kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
     - NOTE: (If Calico Pod network don't work then use -> Weave Net) [i.e- In Step.7 you still see Status as Not Ready]
     - $ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
   - Verify Network
     - \$ kubectl get pods --all-namespaces
     - \$ kubectl get nodes (Check if the worker nodes are listed)

7. [Step-7] Join Worker nodes with -> Master node

- (To Connect all Worker Nodes to Master Node) [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes]
  - (Paste the kubeadm join command with token copied earlier in All WORKER NODES not in Master Node)
  - (Worker Nodes)
    - \$ kubeadm join 192.168.99.10:6443 --token ftbf8q.giu7pwvel44gqy9s \
      --discovery-token-ca-cert-hash sha256:57b378be50a35f1d8a95ad400aa77438965470bf9013fbf008a70b438b4975f1
- \$ kubectl get nodes (Check if the worker nodes are listed) <-- (Run in Master Node)
- (Make sure all nodes are in STATUS -> Ready state)
- (Master Node)
  - (Verify by running a pod in master)
    - \$ kubectl run nginx --image=nginx
    - \$ kubectl get pods
    - \$ kubectl delete deployment/nginx

---

### Error Faced for above configure steps

**Error** : Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

- Run docker service : \$ systemctl start docker

---

Error: network connection (To make sure 2nd network interface -> static ip address are assigned to VM's)

- \$ ifconfig
- \$ ifup enp0s3 192.168.99.12
- \$ ping www.google.com
- \$ systemctl status network\* (Check network daemon is running fine)
- \$ systemctl status network-online.target
- \$ systemctl restart network-online.target
- \$ ifup enp0s3 192.168.99.12
- (For Redhat)

  - \$ systemctl start NetworkManager
  - \$ systemctl enable NetworkManager
  - \$ nano /etc/sysconfig/network-scripts/ifcfg-enp0s3 (Before editing create .org) ==> [./networks-redhat.sh]

- (or)

- Tried all approach to make the ipAddr static in Redhat, but not working so-
  - \$ yum install network-scripts
  - \$ ifconfig enp0s3 192.168.99.11
  - \$ ifup enp0s3 192.168.99.11

---

**Error** - Worker Nodes are in STATUS -> NotReady state

- First, STEPS to exisiting delete Pod existing POD network - Calico
  - \$ kubeadm reset (Reset kubeadm itself)
  - \$ kubectl get pods --all-namespaces (to verifiy)
  - Steps to re-initalize kubeadm in master node
    - \$ kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.99.10
    - \$ mkdir -p \$HOME/.kube
    - $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    - $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    - [Run this command in Worker Nodes] -> \$ kubeadm join 192.168.99.10:6443 --token 8lbhei.jn6pvxj0bisciryj \
      --discovery-token-ca-cert-hash sha256:59f0fa0c2080713fc31c8fee3d609b2bcea6a54778acb7ad82eadceba1d1aefa
    - \$ kubectl get pods --all-namespaces
    - \$ kubectl get nodes
    - \$ kubeadm token list (To know the token of Master node)
- (If Calico Pod network don't work then use -> Weave Net) [Run in Master Mode]
  - $ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  - (If want to reset wave pod network then) -> weave reset
- (Now add join the worker node with master node via new token)
  - \$ kubeadm reset
  - \$ systemctl stop kubelet
  - $ docker stop -f $(docker ps -aq)
  - $ docker rm -f $(docker ps -aq)
  - $ docker rmi -f $(docker images -q)
  - \$ rm -f /etc/kubernetes/kubelet.conf (This file -> kubelet.conf has details of client token)
  - \$ rm -rf ~/.kube
  - \$ systemctl start kubelet
  - \$ systemctl start docker
  - \$ docker image ls -a (Check no images or container present in worker node)
  - \$ docker container ls -a
  - \$ kubeadm join 192.168.99.10:6443 --token 8lbhei.jn6pvxj0bisciryj \
     --discovery-token-ca-cert-hash sha256:59f0fa0c2080713fc31c8fee3d609b2bcea6a54778acb7ad82eadceba1d1aefa

* Ref:

- https://stackoverflow.com/questions/53040663/kubectl-get-nodes-shows-notready
- https://stackoverflow.com/questions/56493883/how-to-join-worker-node-in-existing-cluster
- https://computingforgeeks.com/join-new-kubernetes-worker-node-to-existing-cluster/
- https://www.serverlab.ca/tutorials/containers/kubernetes/how-to-add-workers-to-kubernetes-clusters/

---

- Cleaning Worker Node
  - \$ kubeadm reset
  - \$ systemctl stop kubelet
  - \$ systemctl stop docker
  - \$ rm -rf /var/lib/cni/
  - \$ rm -rf /var/lib/kubelet/\*
  - \$ rm -rf /etc/cni/
  - \$ rm -f /etc/kubernetes/kubelet.conf
  - \$ rm -rf ~/.kube
  - \$ systemctl start docker
  - \$ systemctl enable docker
  - \$ yum remove kubeadm kubectl kubelet kubernetes-cni kube\*
  - \$ yum autoremove
- Reinstall Kube-related tools back in Worker Node [Step-4]
  - \$ sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
  - \$ sudo systemctl enable --now kubelet

---
