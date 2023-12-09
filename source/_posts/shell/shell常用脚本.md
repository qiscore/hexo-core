---
title: shell常用脚本
date: 2023-12-09 14:32:11
tags: shell
categories: shell
---

# 1. 安装java环境

## 1. 准备jdk安装包，environment.txt 环境变量配置文件

```shell


export JAVA_HOME=/usr/local/software/jdk1.8.0_231
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JRE_HOME=$JAVA_HOME/jre


```

## 2. 准备脚本jdk-install.sh

```shell

#!/bin/bash

echo "解压jdk"
tar -zxvf jdk-8u231-linux-x64.tar.gz

cat environment.txt >> ~/.bash_profile
source ~/.bash_profile
java -version


```

## 3. 执行脚本

```shell
sh jdk-install.sh

```

# 2. 安装nginx

## 1. 准备脚本nginx-install.sh

```shell

#!/bin/bash

software_dir=/usr/local/software
nginx_dir=$software_dir/nginx
mkdir -p $nginx_dir
chmod -R 755 $nginx_dir

cd $nginx_dir 

echo "安装依赖包"
yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel wget
wget http://nginx.org/download/nginx-1.23.2.tar.gz

tar -xvf nginx-1.23.2.tar.gz

cd $nginx_dir/nginx-1.23.2

echo "安装SSL模块"
./configure --prefix=$nginx_dir --with-http_stub_status_module --with-http_ssl_module

make
make install


echo "创建启动脚本"

tee $nginx_dir/startup.sh <<-'EOF'

software_dir=/usr/local/software
nginx_dir=$software_dir/nginx

$nginx_dir/sbin/nginx -c $nginx_dir/conf/nginx.conf

EOF

chmod 755 $nginx_dir/startup.sh

echo "创建停止脚本"
tee $nginx_dir/stop.sh <<-'EOF'

software_dir=/usr/local/software
nginx_dir=$software_dir/nginx

$nginx_dir/sbin/nginx -s stop

EOF
   
chmod 755 $nginx_dir/stop.sh


echo "创建重启脚本"
tee $nginx_dir/restart.sh <<-'EOF'

sh stop.sh
sh startup.sh

EOF

chmod 755 $nginx_dir/restart.sh
#在文件首行插入内容
sed -i '1i\#!/bin/bash' $nginx_dir/*.sh

mv $software_dir/nginx-install.sh $nginx_dir

echo "开放tcp/ip端口80-25535"
firewall-cmd --zone=public --add-port=80-25535/tcp --permanent
firewall-cmd --reload

```

## 2 . 执行脚本

```shell
sh nginx-install.sh

```


# 3. 安装docker

## 1.  准备脚本docker-install.sh

```shell

#!/bin/bash
sudo yum remove docker*
echo "安装必要软件包"

sudo yum install -y yum-utils device-mapper-persistent-data lvm2

echo "Step 2: 添加软件源信息"
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
echo "更新并安装Docker-CE-20.10.17"
sudo yum makecache fast
sudo yum -y install docker-ce-20.10.12-3.el7
echo "设置开机启动"
systemctl enable docker --now
echo "配置镜像加速器"
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
 "registry-mirrors": ["https://y96pbacg.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
#更新系统时间
echo "同步系统时间"
yum install ntpdate -y
ntpdate cn.pool.ntp.org


```

## 2. 执行脚本

```shell
sh docker-install.sh

```

# 4. 安装k8s集群

## 1 . 安装docker
略

## 2. 规划集群节点

主节点master：172.20.10.2
从节点slave：172.20.10.4


## 3. 准备脚本k8s-install.sh

```shell

#!/bin/bash
echo "--------------------step-1 初始化环境-------------------------"
echo "设置节点的hostname"
node_hostname=$1
hostnamectl set-hostname $node_hostname

echo "将 SELinux 设置为 permissive 模式（相当于将其禁用）"
# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo "关闭swap"
#关闭swap
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab

echo "允许 iptables 检查桥接流量"
#允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

#放行端口
sudo tee ./turnon-port.sh <<-'EOF'
#打开端口
#!/bin/bash
echo "Open firewall port $1"
firewall-cmd --zone=public --add-port=$1/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-ports
EOF
chmod 755 ./turnon-port.sh
sh ./turnon-port.sh 30000-32796
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
systemctl restart docker
#更新系统时间
echo "同步系统时间"
yum install ntpdate -y
ntpdate cn.pool.ntp.org

rm -rf /etc/containerd/config.toml
systemctl restart containerd

echo "--------------------step-2 安装kubelet、kubeadm、kubectl-------------------------"

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF


sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes
echo "设置kubelet开机启动"
sudo systemctl enable --now kubelet


echo "--------------------step-3 使用kubeadmin引导集群， 下载各个机器需要的镜像-------------------------"
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
docker pull registry.aliyuncs.com/google_containers/$imageName
done
EOF
   
chmod +x ./images.sh && ./images.sh

echo "所有机器添加master域名映射"
echo "172.20.10.2  cluster-endpoint" >> /etc/hosts
echo "172.20.10.2  master" >> /etc/hosts
echo "172.20.10.4  slave" >> /etc/hosts


current_hostname=$(hostname)

if [ "${current_hostname}"x = "master"x ];then
	echo "主节点初始化"
	kubeadm init \
	--apiserver-advertise-address=172.20.10.2 \
	--control-plane-endpoint=cluster-endpoint \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version v1.20.9 \
	--service-cidr=10.96.0.0/16 \
	--pod-network-cidr=192.168.0.0/16
	
	echo "设置kube config"
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	
	echo "安装网络插件"
	curl https://docs.projectcalico.org/v3.20/manifests/calico.yaml -O
	kubectl apply -f calico.yaml
	
	echo "请在work节点执行kubeadmin join命令加入节点"	
fi


```

## 4.  主节点执行脚本进行初始化集群

` sh k8s-install.sh master `

## 5.  从节点执行脚本初始化集群，并加入主节点的集群(执行主节点初始化结果，有个命令，拷贝过来，在从节点初始化后执行)

` sh k8s-install.sh slave `

# 5. 安装nfs

## 1.  规划nfs节点

server节点: 172.20.10.2

client节点: 172.20.10.4

## 2.  准备安装脚本nfs-install.sh, 可同时用于server端和client端

```shell
#/bin/bash

echo "安装依赖nfs-utils nfs-common："
yum install -y nfs-utils nfs-common

FILE_PATH=/nfs/data
SERVER_TYPE=$1

if [ "${SERVER_TYPE}"x = "server"x ];then
	#nfs服务节点
	echo "$FILE_PATH *(insecure,rw,sync,no_root_squash)" > /etc/exports
	mkdir -p /nfs/data
	systemctl enable rpcbind --now
	systemctl enable nfs-server --now
	systemctl start rpcbind
	systemctl start nfs-server
	#配置生效
	exportfs -r
fi

if [ "${SERVER_TYPE}"x = "client"x ];then
    SERVER_IP=$2
	#nfs客户端节点
	showmount -e $SERVER_IP

	#执行以下命令挂载 nfs 服务器上的共享目录到本机路径 /root/nfsmount
	mkdir -p $FILE_PATH
	mount -t nfs $SERVER_IP:$FILE_PATH $FILE_PATH
	#验证挂载生效
	# 写入一个测试文件
	echo "hello nfs server" > $FILE_PATH/example.txt
	echo "please to the nfs server of $SERVER_IP read $FILE_PATH/example.txt"
fi

```

## 3. 安装server端，执行脚本
```shell

sh nfs-install.sh server

```

## 4. 安装client端，指定server的ip地址，执行脚本
```shell

sh nfs-install.sh client 172.20.10.2

```