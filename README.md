基本思路：

VMware创建三台虚拟机，利用docker，kubelet，kubeadm搭建一个cluster，calico和nfs负责网络连接。kubernetes在master node管理下面的两个子节点。用jmeter进行压力测试，hpa监控指标并根据设置自动扩容

1. # 参考

   https://github.com/Youngpig1998/KuberneteCluster-built

   ==先calico，再nfs，再mysql- operator==

   | Steps                      | Manuals                                                      |
   | -------------------------- | ------------------------------------------------------------ |
   | Pre-request                | [pre-request](https://github.com/HenryVarro666/KuberneteCluster-built/blob/main/pre-request) |
   | Use kubeadm to install k8s | [kubeadm](https://github.com/HenryVarro666/KuberneteCluster-built/blob/main/kubeadm) |
   | Use binary to install k8s  | [binary](https://github.com/HenryVarro666/KuberneteCluster-built/blob/main/binary) |
   | Deploy Kubesphere in k8s   | [kubesphere](https://github.com/HenryVarro666/KuberneteCluster-built/blob/main/kubesphere) |

   # 搭建三台虚拟机

   本操作是在Windows上操作的，用VMware Workstation进行虚拟机搭建,操作系统是Linux的CentOS 7 64-bit。


   ## VMware Station设置
   New Virtual Machine （ctrl + N）
   ---> 选择 Custom(advanced) 
   ---> Hardware compatibility 【默认】
   ---> I will install the operating system later 
   ---> Guest operating system: **Linux**  / Version: **CentOS 7 64-bit** 
   ---> Name + Location ---> Processors (这里我选择的是2 * 2) 
   ---> Memory我选择8 GB（因为我desktop本身32G内存）
   ---> Use network address translation (NAT) 【默认】
   ---> I/O Controller types: LSI Logic 【默认】
   ---> Virtual Disk type: SCSI【默认】
   ---> Disk: Create a new virtual disk
   ---> Maximum disk size(GB): 50 GB 【本身推荐的是20GB】；Split virtual disk into multiple files 【默认】
   ---> Disk file: [Name].vmdk 【默认】
   ---> Custiomize Hardware里面New CD/DVD(IDE)的Connection选择Use ISO Image file：**CentOS-7-x86_64-DVD-2009.iso**
   ---> Finish

   至此就可以Power On打开进入设置了

---
   ## CentOS设置
   开机界面Install CentOS 7
   选择语言（图方便选了Simplified Chinese）

   软件
   	软件选择：啥都不懂建议左侧选择GNOME桌面，右边不用选；可以了选完成

   系统
   	安装位置：重新点击选择本地标准磁盘就可以了，可以了选完成
   	禁用KDUMP：不知道是啥，先关了
   	网络和主机名（N）：打开连接
   	
   开始安装
   用户设置就随便设置下，记住管理员密码

   等待1407个部分安装完毕，重启然后使用

   ### 重启之后
   初始设置：接受许可证
   完成配置 
   输入密码即可登录

   进入选择语言：汉语 ---> 汉语
   ---> 隐私：开【默认】
   ---> 连接您的在线账号：跳过【默认】
   ---> 开始使用 CentOS Linux（S）

   至此基本安装完毕

   ### 重要：
   1. 接下来会手动升级到指定版本。
      虽然系统提示有可用的升级，但是
      不要升级！
      不要升级！
      不要升级！

   2. 创建完第一个虚拟机之后可以手动创建第二个，也可以选择clone; create a full clone

   	直接克隆的虚拟机需要手动修改hostname和IP地址

   ## CentOS环境
   `su`指令切换root
   ### 升级内核
   ```shell
   #导入ELRepo仓库的公共密钥
   rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org  
   #安装ELRepo仓库的yum源
   rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm  
   #查看可用的系统内核包
   yum --disablerepo="*" --enablerepo="elrepo-kernel" list available  
   #安装最新版本内核
   yum --enablerepo=elrepo-kernel install kernel-ml  
   
   grub2-set-default 0
   
   #生成 grub 配置文件并重启
   grub2-mkconfig -o /boot/grub2/grub.cfg  
   
   reboot
   
   
   # 从3升级到5了。
   # 如果自动升级到7，kubelet可能会出错
   ```

   #### error
   > Another app is currently holding the yum lock; waiting for it to exit...

   解决方法：
   1. 查进程
   ```shell
    ps -ef | grep xxx
   ```
   2. 杀进程
   ```shell
    kill -9 xxx
    #或者
    pkill PackageKit
   ```

   ### 关闭防火墙
   ```shell
   systemctl stop firewalld
   systemctl disable firewalld
   ```

   ### 关闭selinux
   ```shell
   sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
   setenforce 0  # 临时
   ```

   ### 关闭swap
   ```shell
   swapoff -a  # 临时
   vim /etc/fstab  # 永久,进入后注释掉有swap的那一行
   ```
   > 新的linux版本好像说是 不关闭swap可能会自动杀进程

   ### 设置主机名
   ```shell
   hostnamectl set-hostname <hostname>
   ```

   ### 检查本机ip地址
   ```shell
   ifconfig
   ```

   ### 在master添加hosts
   PS：node节点可以不需要添加hosts
   ```shell
   cat >> /etc/hosts << EOF
   192.168.177.137 Test1
   192.168.177.138 Test2
   192.168.177.139 Test3
   EOF
   ```
   > cat >> /etc/hosts << EOF
   > 192.168.177.134 record0
   > EOF

   ### 将桥接的IPv4流量传递到iptables的链
   ```shell
   cat > /etc/sysctl.d/k8s.conf << EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   
   # 生效
   sysctl --system  
   ```

   ### 时间同步
   必须同步才行
   ```shell
   yum install ntpdate -y
   ntpdate time.windows.com
   ```

   ### 配置==静态ip==地址
   ```shell
   ifconfig   #查看IP地址
   
   vim /etc/sysconfig/network-scripts/ifcfg-ens33 
   TYPE="Ethernet"
   PROXY_METHOD="none"
   BROWSER_ONLY="no"
   BOOTPROTO="static"    #这里要修改为static   
   DEFROUTE="yes"
   IPV4_FAILURE_FATAL="no"
   IPV6INIT="yes"
   IPV6_AUTOCONF="yes"
   IPV6_DEFROUTE="yes"
   IPV6_FAILURE_FATAL="no"
   IPV6_ADDR_GEN_MODE="stable-privacy"
   NAME="ens33"
   UUID="cd4ce59b-0bcf-42b8-ab7d-312644bb46f3"
   DEVICE="ens33"
   ONBOOT="yes"
   IPADDR="192.168.177.137"    #从这一行开始都是要添加的，这里添加上述查看到的ip地址
   PREFIX="24"
   GATEWAY="192.168.177.2"    #需要与ip地址相对应
   DNS1="8.8.8.8"   #我这里是我学校的DNS，你可以自己选择一个公有DNS
   ```
   > GATEWAY 和ip address前面一样，最后是2

   ### 克隆的虚拟机修改ipv4地址(可以等到之后的软件都装完了再克隆)
   ==不要忘记修改 hostname==
   和上面一样的文件
   ```shell
   vim /etc/sysconfig/network-scripts/ifcfg-ens33 
   
   systemctl restart network
   
   hostnamectl set-hostname <hostname>
   
   ifconfig
   
   ```

   ### 重启网络服务
   ```shell
   systemctl restart network
   ping www.baidu.com
   ```
   ping通就说明配置好了


   # 使用kubeadm安装部署

   > 版本可依据个人情况，本文使用的是**1.20.0**

   kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

   > kubeadm能通过两条指令完成一个kubernetes集群的部署
   > 比如
   ```shell
   # 创建一个 Master 节点
   kubeadm init

   # 将一个 Node 节点加入到当前集群中
   kubeadm join <Master节点的IP和端口 >
   ```

   ## 1. 安装Docker/kubeadm/kubelet【==所有节点==】
   ```shell
   echo 1 > /proc/sys/net/ipv4/ip_forward # 每个节点可能还需要用一下这个
   ```
   Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

   ### 1.1 安装Docker
   注意版本与Kubernetes版本的兼容性

   ```shell 
   wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
   yum install -y docker-ce-19.03.5-3.el7 docker-ce-cli-19.03.5-3.el7 containerd.io
   systemctl enable docker && systemctl start docker
   ```

   #### 配置镜像下载加速器
   ```shell
   cat > /etc/docker/daemon.json << EOF
   {
     "registry-mirrors": ["https://tzksttqp.mirror.aliyuncs.com"]
   }
   EOF
   ```

   #### 修改cgroup driver
   ```shell
   vim /usr/lib/systemd/system/docker.service
   #在ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock  这一行添加 --exec-opt native.cgroupdriver=systemd  参数
   
   systemctl daemon-reload &&  systemctl restart docker
   
   
   docker info | grep Cgroup   #查看docker的 cgroup driver
   ```
   > 1. 每次修改配置都需要 `systemctl daemon-reload &&  systemctl restart XX`
   > 2. Cgroup driver修改成systemd，保持一致

   ### 1.2 添加阿里云YUM软件源，为了能够下载kubeadm、kubelet和kubectl
   ```shell
   cat > /etc/yum.repos.d/kubernetes.repo << EOF
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   ```

   ### 1.3 安装kubeadm，kubelet和kubectl
   由于版本更新频繁，这里可以**指定版本号部署**：（不指定即下载最新版本），这里使用的是**1.20**版本
   ```shell
   yum install -y kubelet-1.20.0 kubeadm-1.20.0 kubectl-1.20.0
   
   
   systemctl enable kubelet
   ```
   > 不指定上面的软件源可能找不到这里所需要的版本号

   ## 部署Kubernetes Master
   参考文档：
   1. [https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)
   2. [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node)
   ### 在Master节点执行
   ```shell
   kubeadm init --apiserver-advertise-address=192.168.177.134 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.20.0 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all
   ```

   ==--pod-network-cidr=10.244.0.0/16==

   #### 或者使用配置文件引导（推荐）
   ```shell 
   vi kubeadm.conf
   
   apiVersion: kubeadm.k8s.io/v1beta2
   kind: ClusterConfiguration
   kubernetesVersion: v1.20.0
   imageRepository: registry.aliyuncs.com/google_containers 
   networking:
     podSubnet: 10.244.0.0/16 
     serviceSubnet: 10.96.0.0/12 
   
   # 设置完kubeadm之后执行kubeadm init
   kubeadm init --config kubeadm.conf --ignore-preflight-errors=all
   ```
   ==此处的podSubnet设置在之后的Calico里面有用到==


   ```shell
   [root@record0 ccao2679]# kubeadm init --config kubeadm.conf --ignore-preflight-errors=all
   [init] Using Kubernetes version: v1.20.0
   [preflight] Running pre-flight checks
   [preflight] Pulling images required for setting up a Kubernetes cluster
   [preflight] This might take a minute or two, depending on the speed of your internet connection
   [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
   [certs] Using certificateDir folder "/etc/kubernetes/pki"
   [certs] Generating "ca" certificate and key
   [certs] Generating "apiserver" certificate and key
   [certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local record0] and IPs [10.96.0.1 192.168.177.134]
   [certs] Generating "apiserver-kubelet-client" certificate and key
   [certs] Generating "front-proxy-ca" certificate and key
   [certs] Generating "front-proxy-client" certificate and key
   [certs] Generating "etcd/ca" certificate and key
   [certs] Generating "etcd/server" certificate and key
   [certs] etcd/server serving cert is signed for DNS names [localhost record0] and IPs [192.168.177.134 127.0.0.1 ::1]
   [certs] Generating "etcd/peer" certificate and key
   [certs] etcd/peer serving cert is signed for DNS names [localhost record0] and IPs [192.168.177.134 127.0.0.1 ::1]
   [certs] Generating "etcd/healthcheck-client" certificate and key
   [certs] Generating "apiserver-etcd-client" certificate and key
   [certs] Generating "sa" key and public key
   [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
   [kubeconfig] Writing "admin.conf" kubeconfig file
   [kubeconfig] Writing "kubelet.conf" kubeconfig file
   [kubeconfig] Writing "controller-manager.conf" kubeconfig file
   [kubeconfig] Writing "scheduler.conf" kubeconfig file
   [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [kubelet-start] Starting the kubelet
   [control-plane] Using manifest folder "/etc/kubernetes/manifests"
   [control-plane] Creating static Pod manifest for "kube-apiserver"
   [control-plane] Creating static Pod manifest for "kube-controller-manager"
   [control-plane] Creating static Pod manifest for "kube-scheduler"
   [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
   [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
   [apiclient] All control plane components are healthy after 6.001957 seconds
   [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
   [kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
   [upload-certs] Skipping phase. Please see --upload-certs
   [mark-control-plane] Marking the node record0 as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
   [mark-control-plane] Marking the node record0 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
   [bootstrap-token] Using token: xe5sop.b3kcpevue4v07uqh
   [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
   [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
   [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
   [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
   [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
   [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
   [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
   [addons] Applied essential addon: CoreDNS
   [addons] Applied essential addon: kube-proxy
   
   Your Kubernetes control-plane has initialized successfully!
   
   To start using your cluster, you need to run the following as a regular user:
   
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
   Alternatively, if you are the root user, you can run:
   
     export KUBECONFIG=/etc/kubernetes/admin.conf
   
   You should now deploy a pod network to the cluster.
   Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
     https://kubernetes.io/docs/concepts/cluster-administration/addons/
   
   Then you can join any number of worker nodes by running the following on each as root:
   
   kubeadm join 192.168.177.134:6443 --token xe5sop.b3kcpevue4v07uqh \
       --discovery-token-ca-cert-hash sha256:02ed0d4c841e8905aa4e3b6e65ca5647db526732f6276c3ae074d7d97fb334af 
   
   ```


   > 可能会出现It seems like the kubelet isn't running or healthy的错误, 建议参考
   > 1. [https://blog.csdn.net/boling_cavalry/article/details/91306095](https://blog.csdn.net/boling_cavalry/article/details/91306095)
   > 2. [https://blog.csdn.net/weixin_41298721/article/details/114916421](https://blog.csdn.net/weixin_41298721/article/details/114916421)
   > 3. [http://www.manongjc.com/detail/23-umxjtqmublnyjwl.html](http://www.manongjc.com/detail/23-umxjtqmublnyjwl.html)

   -   --apiserver-advertise-address 集群通告地址
   -   --image-repository 由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址
   -   --kubernetes-version K8s版本，与上面kubeadm等安装的版本一致
   -   --service-cidr 集群内部虚拟网络，Pod统一访问入口
   -   --pod-network-cidr Pod网络，，与下面部署的CNI网络组件yaml中保持一致
   -   --ignore-preflight-errors=all 忽视一些不是很重要的警告

   ### kubeadmin init步骤
   1、[preflight] 环境检查

   2、[kubelet-start] 准备kublet配置文件并启动/var/lib/kubelet/config.yaml

   3、[certs] 证书目录 /etc/kubernetes/pki

   4、[kubeconfig] kubeconfig是用于连接k8s的认证文件

   5、[control-plane] 静态pod目录 /etc/kubernetes/manifests 启动组件用的

   6、[etcd] etcd的静态pod目录

   7、[upload-config] kubeadm-config存储到kube-system的命名空间中

   8、[mark-control-plane] 给master节点打污点，不让pod分配

   9、[bootstrp-token] 用于引导kubernetes的证书

   ### 拷贝kubectl使用的连接k8s认证文件到默认路径
   ```shell
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config
   kubectl get nodes
   
   #  NAME         STATUS   ROLES    AGE   VERSION
   #  k8s-master   Ready    master   2m    v1.20.0
   ```

   

   ### 错误
   1. The connection to the server localhost:8080 was refused - did you specify the right host or port?

   原因：
   kubernetes master没有与本机绑定，需要设置环境变量

   解决方法：
   ```shell
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
       
   source /etc/profile
   ```

   2.  Config not found: /etc/kubernetes/admin.conf The connection to the server localhost:8080 was refused - did you specify the right host or port?
      解决办法：
      kubeadm init之后才好使；可能需要等一等，等pod起来


   ## 加入Kubernetes Node
   在子节点执行。

   ### 向集群添加新节点，在子node结点上执行：
   ```shell
   kubeadm join 192.168.177.134:6443 --token xe5sop.b3kcpevue4v07uqh --discovery-token-ca-cert-hash sha256:02ed0d4c841e8905aa4e3b6e65ca5647db526732f6276c3ae074d7d97fb334af 
   ```

   默认token有效期为24小时，当过期之后，上述token就不可用了,这时就需要重新创建token。

   ### 重新创建token，在master节点中操作
   ```shell
   kubeadm token create
   kubeadm token list    
   openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
   ```

   ### 在node结点中使用最新的token
   ```shell
   kubeadm join 192.168.177.134:6443 --token XXX --discovery-token-ca-cert-hash sha256:XXX
   ```
   > kubeadm token create --print-join-command 可用于生成kubeadm join命令
   > [https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)


---
   #### 报错
   > [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1

   解决方法：
   > https://stackoverflow.com/questions/55531834/kubeadm-fails-to-initialize-when-kubeadm-init-is-called

   ```shell
   echo 1 > /proc/sys/net/ipv4/ip_forward
   ```

   


   ## ==先calico，再nfs，再mysql- operator==


   ## 4.部署容器网络（CNI）
   [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

   Flannel或者Calico，这里推荐Calico

   ==**Calico安装在Master节点就可以了**==

   ### 4.1 Calico（推荐）
   Calico是一个纯三层的数据中心网络方案，Calico支持广泛的平台，包括Kubernetes、OpenStack等。

   Calico 在每一个计算节点利用 Linux Kernel 实现了一个高效的虚拟路由器（ vRouter） 来负责数据转发，而每个 vRouter 通过 BGP 协议负责把自己上运行的 workload 的路由信息向整个 Calico 网络内传播。

   此外，Calico 项目还实现了 Kubernetes 网络策略，提供ACL功能。

   #### 官方下载，自己改参数
   [https://docs.projectcalico.org/getting-started/kubernetes/quickstart](https://docs.projectcalico.org/getting-started/kubernetes/quickstart)

   ```shell
   wget https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate
   ```

   下载完后还需要修改里面配置项：
   -   定义Pod网络（CALICO_IPV4POOL_CIDR），==与前面kubeadmin.conf文件中的podSubnet配置一样==
   -   选择工作模式（CALICO_IPV4POOL_IPIP），支持**BGP（Never）**、**IPIP（Always）**、**CrossSubnet**（开启BGP并支持跨子网）

   ![calico](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208231316257.png)


   #### 使用现成的yaml文件（推荐）
   也需要修改CIDR部分
   https://github.com/Youngpig1998/kubernetes-tutorial/tree/main/workspace/lesson0/kubeadm

   修改完后应用清单:
   ```shell
   kubectl apply -f calico.yaml
   
   kubectl get pods -n kube-system
   ```


   ### NFS
   #### 1. nfs文件系统
   ##### 安装nfs-server

   ```shell
   yum install -y nfs-utils 
   yum install -y rpcbind
   
   systemctl start rpcbind    #先启动rpc服务
   systemctl enable rpcbind   #设置开机启动
   
   systemctl start nfs-server    
   systemctl enable nfs-server
   
   echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports
   
   # 执行以下命令，启动 nfs 服务;创建共享目录
   mkdir -p /nfs/data
   
   # 使配置生效
   systemctl reload nfs
   ```

   #### 2. 配置storageclass资源
   配置动态供应的默认存储类

   经测试，在1.20版本下该nfs-provisioer无法正常运行，通过log查看日志后显示报错信息“selfLink was empty”（也可能是其他的），这是由于selfLink在1.16版本以后已经弃用，在1.20版本停用。
   而由于nfs-provisioner的实现是基于selfLink功能（同时也会影响其他用到selfLink这个功能的第三方软件），需要等nfs-provisioner的制作方重新提供新的解决方案。目前可用的临时方案是：
   ```shell
    vim /etc/kubernetes/manifests/kube-apiserver.yaml #文件，找到如下内容后，在最后添加一项参数 
    
   spec:
   	containers:
   	- command:
       - kube-apiserver    
       - --advertise-address=192.168.210.20    
   	- --.......　　#省略多行内容    
       - --feature-gates=RemoveSelfLink=false　　#添加此行
    #如果是高可用的k8s集群，则需要在所有master节点上进行此操作。
    #稍等片刻后kube-apiserver的pod会自动重启。也可以自己手动删除pod
   ```

   ### 5个yaml文件
   https://github.com/Youngpig1998/kubernetes-tutorial/tree/main/workspace/kubesphere  下面的storageclass文件夹

   需要修改deployment-nfs.yaml中的内容
   1. 指定nfs-server的IP地址 
   2. 修改路径为 /nfs/data

   ```shell
   # 需要修改deployment-nfs.yaml中的内容，指定nfs-server的IP地址 + 修改路径为 /nfs/data
   
   kubectl apply -f storageclass/clusterrole.yaml
   kubectl apply -f storageclass/clusterrolebinding.yaml
   kubectl apply -f storageclass/serviceaccount.yaml
   kubectl apply -f storageclass/deployment-nfs.yaml
   kubectl apply -f storageclass/nfs-class.yaml
   
   #确认配置是否生效
   kubectl get sc
   
   # 结果
   # NAME                    PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   # nfs-storage (default)   fuseim.pri/ifs   Delete          Immediate           false                  78s
   
   ```

   ## 项目下载
   ### 1. 安装git
   ```shell
   yum install git
   ```
   git clone 
   ```shell
   git clone https://github.com/ringdrx/visitors-operator.git
   ```

   ### 2. 安装go
   [How to Install Golang on CentOS 7?](https://www.geeksforgeeks.org/how-to-install-golang-on-centos-7/)
   #### 下载压缩包
   ```shell
   wget https://dl.google.com/go/go1.19.linux-amd64.tar.gz
   ```
   > 也可以官网下载，但是这边用1.19版本

   #### 解压文件
   ```shell
   tar -xf go1.19.linux-amd64.tar.gz
   ```

   #### 修改profile文件
   `pwd`查看当前路径，我的是root路径下有个go文件夹
   ```shell
   vim /etc/profile
   
   # 在里面加上export PATH=$PATH:/root/go/bin
   ```
   > 或者 `export PATH=$PATH:/root/go/bin`

   #### 实现profile设置
   ```shell
   source /etc/profile
   ```

   #### 检查
   ```shell
   go version
   ```
   显示`go version go1.19 linux/amd64`，成功

   ### 3. 安装gcc
   ```shell
   yum install gcc
   ```

   ### 4. 安装helm
   #### 下载压缩包
   ```shell
   wget https://get.helm.sh/helm-v3.6.0-linux-amd64.tar.gz
   ```
   > 这边用3.6版本

   #### 解压文件
   ```shell
   tar -xf helm-v3.6.0-linux-amd64.tar.gz
   ```

   #### 移动文件夹
   ```shell
   mv linux-amd64/helm /usr/local/bin
   ```

   #### 删除不必要文件
   ```shell
   rm -rf helm-v3.6.0-linux-amd64.tar.gz
   
   rm -rf linux-amd64
   ```

   #### 检查
   ```shell
   helm version
   ```
   显示`version.BuildInfo{Version:"v3.6.0", GitCommit:"7f2df6467771a75f5646b7f12afb408590ed1755", GitTreeState:"clean", GoVersion:"go1.16.3"}`，成功

   #### 检查ip
   ```shell
   #### 显示ip
   -o: This option will list more information, including the node the pod resides on, and the pod's cluster IP. The IP column will contain the internal cluster IP address for each pod
   ```shell
   # 输出node信息
   kubectl get nodes -o wide
   ```

   ## 运行项目
   ### Step 1: helm安装mysql-operator
   #### 添加repo
   ```shell
   helm repo add bitpoke https://helm-charts.bitpoke.io
   ```

   #### 修改
   ```shell
   helm pull bitpoke/mysql-operator --untar
   ```

   > 之前已经：安装nfs-server + deployment-nfs.yaml： 配置storageclass （5个文件）
   > 在operator里的values.yaml配置storageclass的名字
   > ![storage clasa neme](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208081236571.png)


   解决：
   1. helm pull把mysql-operator仓库拉到本地
   2. 修改里面的values.yaml文件，在文件中把storageclass那一项取消注释
   3. "-" 修改成nfs-class.yaml （kind:StorageClass)里面name的名字：nfs-storage


   ### 安装mysql-operator
   ```shell
   # helm install mysql-operator bitpoke/mysql-operator
   
   # helm install mysql-operator presslabs/mysql-operator
   
   # helm uninstall mysql-operator 
   
   helm install mysql-operator ./mysql-operator 
   ```

   ==mysql-operator有可能会先失败几次，然后自己就好了==
   ![mysql-operator 启动](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208200308184.png)


   ==重启之后需要等待一段时间，等里面的pods都起来了才可以get nodes==
   ### 待解决问题
   有一些pods没起来（可能虽然不影响整个程序的运行），
   比如controller和scheduler
   ![一部分pods无法正常启动](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208222259783.png)


   ### Step 2: Create MySQL cluster 
   文件在visitors-operator项目文件夹里
   ```shell
   # create the database’s secret
   kubectl apply -f config/samples/mysql/example-cluster-secret.yaml
   # kubectl delete -f config/samples/mysql/example-cluster-secret.yaml
   
   # create the database cluster
   kubectl apply -f config/samples/mysql/example-cluster.yaml
   # kubectl delete -f config/samples/mysql/example-cluster.yaml
   ```


   # 安装 Visitor Operator
   ## Intellij Idea监控运行
   ### xftp
   为了方便Windows和虚拟机里的linux交互和发送文件，可以下载一个[xftp](https://www.netsarang.com/en/xftp/)

   New（Ctrl+N）
   ---> host:输入主机ip，user name + password
   ---> Connect后accept and save 就可以了

   ==用户名选root==

   > 打开Tools ---> options，下面General里面有个Show hidden files，选择了就可以看到./kube之类的隐藏文件

   把./kube文件夹拷贝到windows usr/用户/下面

   ### Intellij Idea
   Windows也要安装go
   #### Idea go设置
   打开idea的settings，Languages & Frameworks下面go里面编辑GOROOT和GOPATH，和环境变量一致即可

   #### Project设置
   选择项目文件夹之后右上角Edit Configurations，选择Go Build

   里面选File
   File是main.go

   >Package
   >Package需要提供一个path,可以是github


   ### 可能出现的报错
   1. ![fail to get api](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208222330728.jpg)
      解决方法：
      把虚拟机里的.kube删了，在重新执行那几条创建.kube的命令，再复制到宿主机上
      可能是./kube没有更新的缘故

   2. ![fail to wait for](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208222331250.jpg)
      解决方法：
      先去宿主机make install
      make install其实就是apply crd资源

   ==先 make generate，再make manifests 再 make install==

   3. 
   > bash: /root/Build-K8s-Cluster/visitors-operator/bin/controller-gen: 没有那个文件或目录


   在visitor opertor文件夹下创建bin文件夹，里面放入controller-gen
   > 虽然bin/下面就是operator自己创建的。里面包含了要用到的二进制文件，比如kustmize之类的。这些是执行operator-sdk时候生成的

   #### 修改文件权限
   从windows复制过来的文件需要修改成可执行的权限
   ```shell
   chmod +x controller-gen
   ```


   ### 新建Visitors-operator 文件夹之后

   想要修改文件名的话，可以直接 `mv <旧文件名> <新文件名>`

   下载operator-sdk,并修改成可执行权限并放到/usr/local/bin文件夹下

   ```shell
   operator-sdk init --plugins go/v3  --domain my.domain  --repo github.com/ringdrx/visitors-operator
   ```


   ```shell
   operator-sdk create api --group example.com  --version v1beta1 --kind VisitorsApp  --resource --controller
   ```


   #### 需要修改
   1. api文件夹下面的v1beta1里面的visitorsapp_types.go
   2. controllers文件夹
   3. main.go文件

   ```shell
   make generate # visitorsapp_types.go修改了之后 就需要执行make generate
   ```

   ```shell
   cp /root/Build-K8s-Cluster/visitors-operator/api/v1beta1/visitorsapp_types.go .
   ```

   ### Make Install （==在master 节点执行==）
   ```shell
   #项目文件夹下安装CRD
   make install  # 生成一个crd的yaml文件并且部署
   # make uninstall 
   ```

   到这一步位置都是为了创建一个crd

   #### ==bin文件夹里的controller-gen和kustomize文件很重要==
   （第一次上传git这个文件夹和两个文件不见了，很奇怪）

   ### Idea 运行 main.go
   > 用IDE省去很多步骤

   ### Create a Custom Resource (CR)
   ```shell
   kubectl apply -f config/samples/example.com_v1beta1_visitorsapp.yaml
   # kubectl delete -f config/samples/example.com_v1beta1_visitorsapp.yaml
   ```

   


----
   ## 报错
   ### 1. mysql-operator出错message
   >  0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims.

   > mysql-operator会去创建mysql cluster，他们是以statefulset形式运行，storageclass是用来自动创建mysql需要的pv和pvc
   > 出现这个问题一般就是网络那边没搭建好。Calico和NFS检查下

   ### PV & PVC
   Templates/statefulset.yaml各种yaml文件会调用mysql-opertaor/values文件配的各种参数  

   volumeClaimTemplates会创建PV

   ```shell
   >  persistence:
   >  
   >  enabled: true
   > ## If defined, storageClassName: <storageClass>\
   >  ## If set to "-", storageClassName: "", which disables dynamic provisioning
   >  ## If undefined (the default) or set to null, no storageClassName spec is
   >  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
   >  ##   GKE, AWS & OpenStack)
   >  ##
   >  storageClass: "-"
   >  # annotations: {}
   >  # selector:
   >  #  matchLabels: {}
   ```

   ## 2. The connection to the server localhost:8080 was refused - did you specify the right host or port?

   原因：
   1. kubernetes master没有与本机绑定，需要设置环境变量

   解决：
   ```shell
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
       
   source /etc/profile
   ```
   2. pod没起来，检查下再等等

   ## 3. 每次reset之后
   k8s重置，啥都没了。
   拷贝kubectl使用的连接k8s认证文件到默认路径
   ```shell
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config
   ```
   ./kube文件也需要重新复制

   

   ## metrics-server
   复制 metrics-server文件夹到项目文件夹

   安装监控组件
   ```shell
   # 集群指标监控组件，如HPA、VPA等
   kubectl apply -f metrics-server/components.yaml
   ```

   结果
   ```shell
   serviceaccount/metrics-server created
   clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
   clusterrole.rbac.authorization.k8s.io/system:metrics-server created
   rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
   clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
   clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
   service/metrics-server created
   deployment.apps/metrics-server created
   apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
   ```

   

   ```shell 
   kubectl autoscale deploy visitorsapp-sample-frontend  --max=5 --cpu-percent=50
   ```


   ```shell
   kubectl get hpa # 必须和metric server结合使用
   ```


---

   使用metrics server之后使用这条命令可以看到各个节点（工作节点）的指标

   ```shell
    kubectl top nodes # 或者pod
   ```

   # Apache Jemeter
   1. apache-jmeter-5.4.3\bin
      点击jemeter.bat来运行程序
   2. 添加插件
      jmeter-plugins-manager-1.7.tar
      放到apache-jmeter-5.4.3/lib/ext 文件夹下面
   3. 保存test_plan.jmx文件，拖到jemeter里面打开

   [JMeter压力测试步骤](https://blog.51cto.com/u_15192324/2764444?abTest=51cto)

   # Test Result

   原来：

   unknown可能是因为hpa还没起来，也可能是因为网络设置有问题，比如之前calico那个subnet的问题

   前端正常没有负载应该就是0%

   ![Result](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208241847197.png)

   后来：

   ![Test Result](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208241843000.png)

   

## 自动扩容
![压力](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208241906382.png)

![自动扩容](https://raw.githubusercontent.com/HenryVarro666/images/master/imgs/202208241906687.png)

   # To-do

   ### 1. KubeSphere

   > [KubeSphere](https://kubesphere.io/) 是在 [Kubernetes](https://kubernetes.io/) 之上构建的面向云原生应用的**分布式操作系统**，完全开源，支持多云与多集群管理，提供全栈的 IT 自动化运维能力，简化企业的 DevOps 工作流。它的架构可以非常方便地使第三方应用与云原生生态组件进行即插即用 (plug-and-play) 的集成。
   >
   > 作为全栈的多租户容器平台，KubeSphere 提供了运维友好的向导式操作界面，帮助企业快速构建一个强大和功能丰富的容器云平台。KubeSphere 为用户提供构建企业级 Kubernetes 环境所需的多项功能，例如**多云与多集群管理、Kubernetes 资源管理、DevOps、应用生命周期管理、微服务治理（服务网格）、日志查询与收集、服务与网络、多租户管理、监控告警、事件与审计查询、存储管理、访问权限控制、GPU 支持、网络策略、镜像仓库管理以及安全管理**等。
   >
   > KubeSphere 还开源了 [KubeKey](https://github.com/kubesphere/kubekey) 帮助企业一键在公有云或数据中心快速搭建 Kubernetes 集群，提供单节点、多节点、集群插件安装，以及集群升级与运维。

   ### KubeSphere 文档

   https://kubesphere.io/zh/docs/v3.3/

   ### What is KubeSphere

   https://github.com/kubesphere/kubesphere

   ### 2. Kubespray
