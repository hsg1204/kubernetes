Single node cluster 
====
# 安装kubeadm的流程和大部分问题解决方法
https://blog.csdn.net/zzq900503/article/details/81710319<br>https://blog.csdn.net/nklinsirui/article/details/80581286#debian-ubuntu
# 安装kubeadm
        root用户下操作（需要挂代理）
        $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
        $ cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
        deb http://apt.kubernetes.io/ kubernetes-xenial main
        EOF
        $ apt-get update
        $ apt-get install -y docker.io kubeadm
# 经常需要用到的两个docker查看状态的命令（docker重启失败可以运用）：
        $ journalctl -xe
        $ systemctl status docker.service
        多数情况下如果显示是daemon.json文件的问题，则通过更改他的内容来修复，可以参考官方文档：
            https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-storage-driver
        更改/etc/docker/daemon.json
        $ systemctl daemon-reload
        $ systemctl enable docker
        $ systemctl start docker
# 安装完kubeadm以后需要kubeadm init
## 遇到的问题一：swap问题，只需要关闭swap即可，但是还需要看具体的报错情况。
    $ swapoff -a
## 遇到的问题二：[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-apiserver:v1.12.1: output: Error response from daemon: Get https://k8s.gcr.io/v1/_ping: dial tcp 108.177.125.82:443: i/o timeout
            意思是缺少kubeadm init时候需要的镜像，首先需要拉取所需镜像。
            查看所需镜像的版本命令：
            $ kubeadm config images list 
            k8s.gcr.io/kube-apiserver:v1.12.2
            k8s.gcr.io/kube-controller-manager:v1.12.2
            k8s.gcr.io/kube-scheduler:v1.12.2
            k8s.gcr.io/kube-proxy:v1.12.2
            k8s.gcr.io/pause:3.1
            k8s.gcr.io/etcd:3.2.24
            k8s.gcr.io/coredns:1.2.2
            我的kubeadm版本是1.12.2，所需镜像版本和kubeadm版本有关系。拉取镜像可以利用一个脚本一起拉取：
            {
                        待补充
            }
            进入阿里云的镜像平台下载(比较推荐，也可以挂代理拉取，但是需要配置docker的代理)：
                    https://cr.console.aliyun.com/cn-hangzhou/images
            进入网站以后搜索所需镜像找到镜像公网地址加版本号进行拉取（）：
                    $ docker pull registry.cn-hangzhou.aliyuncs.com/v1_12/kube-controller-manager:v1.12.2  
            拉取玩镜像以后个根据所需镜像更改名字：
                    $ docker tag 15e9da1ca195 k8s.gcr.io/kube-proxy:v1.12.2
            这个时候可以运行kubeadm init
            出现：
            ............................
            Your Kubernetes master has initialized successfully!

            To start using your cluster, you need to run the following as a regular user:
            `mkdir -p $HOME/.kube`
            sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            sudo chown $(id -u):$(id -g) $HOME/.kube/config

            You should now deploy a pod network to the cluster.
            Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
               https://kubernetes.io/docs/concepts/cluster-administration/addons/

            You can now join any number of machines by running the following on each node as root:
            kubeadm join 10.12.12.183:6443 --token 57xxux.arjdeme44pfk7yd7 --discovery-token-ca-cert-hash                                   sha256:95c6caf4cdf38931fb79825fc5601e383b1e327f166e7e131e1f17ac39a0ee1a
## 问题三：若开始kubeadm init失败的，再次操作时需要kubeadm reset清理环境重来
在init以后一定要执行高亮的语句，否则可能出现下列问题或者每次都需要导入环境变量来识别加密文件：
            $ kubectl get pods -n kube-system
            Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")

## 问题四：
        发现需要用kubeadm init  --pod-network-cidr=10.244.0.0/16 由于节点之间需要通信这里利用flannel网络，可以参照这个网站：https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
        利用sysctl net.bridge.bridge-nf-call-iptables=1将 /proc/sys/net/bridge/bridge-nf-call-iptables改为1 将桥接的IPv4流量传递给iptables的链然后运行下述命令来安装一个pod网络：
        $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml

        运行结束以后检查kube-system状态：
        $ kubectl get pods -n kube-system            
          coredns一直处于crashloopbackoff的状态
          利用指令查看这两个容器的问题：
        $ kubectl logs -f containername -n kube-system
          .:532018/09/22 07:39:37 [INFO] CoreDNS-1.2.22018/09/22 07:39:37                 
          [INFO] linux/amd64, go1.11, eb51e8b
          CoreDNS-1.2.2
          linux/amd64, go1.11, eb51e8b
          2018/09/22 07:39:37 [INFO] plugin/reload: Running configuration MD5 = f65c4821c8a9b7b5eb30fa4fbc167769
          2018/09/22 07:39:38 [FATAL] plugin/loop: Seen "HINFO IN 87887631643.5216586587165434789." more than twice, loop detected
         查找问题，找到两种解决方法：
         参考网址：http://blog.51cto.com/355665/2178181?source=dra
                 https://github.com/coredns/coredns/issues/2087
                 https://www.jianshu.com/p/08526d0ba398
         利用第二、三个网址方法：
         $ kubectl -n kube-system edit configmap coredns   #将loop注释       
        重新查看状态，coredns不在处于crashloopbackoff状态， 处running状态。
        到这里单节点的kubernetes搭建完成。

# 默认情况下，出于安全原因，群集不会在主服务器上安排容器。如果希望能够在主服务器上安排pod，例如对于单机Kubernetes集群进行开发，运行：
        $ kubectl taint nodes --all node-role.kubernetes.io/master-

# 部署worker节点操作（所有worker节点上安装kubeadm和docker，kubeadm join操作）：
        连上woker节点机器，切换至root用户，
        $ kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha25:<hash>
        获取token和hash：
        $ kubeadm token create
        $ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
           openssl dgst -sha256 -hex | sed 's/^.* //'

# 部署 Dashboard 可视化插件
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
修改 kubernetes-dashboard.yaml文件，不修改直接kubectl apply -f 会出现镜像拉取失败的问题
查看原始yaml文件，查看所需镜像版本，从阿里云镜像库下载
改为 image: registry.cn-hangzhou.aliyuncs.com/wzz/kubernetes-dashboard-amd64:v1.10.0
再create：
$ kubectl create -f kubernetes-dashboard.yaml

若你先操作了kubectl apply -f 则可以
$ kubectl replace --force -f kubernetes-dashboard.yaml

若要删除插件则：
$  kubectl delete -f kube-dashboard.yaml
利用kubectl get pods -n kube-system查看是否配置成功。

# 部分调试用指令
        $ systemctl status kubelet
        $ journalctl -xeu kubelet
        $ journalctl -u docker  
        $ kubectl get services --all-namespaces    查看所有服务
