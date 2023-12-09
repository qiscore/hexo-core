---
title: kubesphere环境安装
date: 2023-12-09 15:20:04
tags: k8s
categories: k8s
---
> 集群规划
master: 172.20.10.2  root/root
slave: 172.20.10.3   root/root


# 1. kubesphere环境初始化

## 1. 准备初始化脚本initial_kube.sh
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
sh ./turnon-port.sh 80-32796
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl restart docker
#更新系统时间
echo "同步系统时间"
yum install ntpdate -y
ntpdate ntp1.aliyun.com

```

## 2. 集群的每个节点执行脚本
` sh initial_kube.sh master`
` sh initial_kube.sh slave`


# 2. kubesphere.yaml配置文件


## 1. 方式1：直接使用以下配置文件

```yaml


apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 172.20.10.2, internalAddress: 172.20.10.2, user: root, password: "root"}
  - {name: slave, address: 172.20.10.3, internalAddress: 172.20.10.3, user: root, password: "root"}
  roleGroups:
    etcd:
    - master
    control-plane: 
    - master
    worker:
    - slave
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    # internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: 1.22.10
    clusterName: cluster.local
    autoRenewCerts: true
    containerManager: docker
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []



---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: 3.3.0
spec:
  persistence:
    storageClass: ""
  authentication:
    jwtSecret: ""
  zone: ""
  local_registry: ""
  namespace_override: ""
  # dev_tag: ""
  etcd:
    monitoring: false
    endpointIps: localhost
    port: 2379
    tlsEnable: true
  common:
    core:
      console:
        enableMultiLogin: true
        port: 30880
        type: NodePort
    # apiserver:
    #  resources: {}
    # controllerManager:
    #  resources: {}
    redis:
      enabled: false
      volumeSize: 2Gi
    openldap:
      enabled: false
      volumeSize: 2Gi
    minio:
      volumeSize: 20Gi
    monitoring:
      # type: external
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
      GPUMonitoring:
        enabled: false
    gpu:
      kinds:
      - resourceName: "nvidia.com/gpu"
        resourceType: "GPU"
        default: true
    es:
      # master:
      #   volumeSize: 4Gi
      #   replicas: 1
      #   resources: {}
      # data:
      #   volumeSize: 20Gi
      #   replicas: 1
      #   resources: {}
      logMaxAge: 7
      elkPrefix: logstash
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchHost: ""
      externalElasticsearchPort: ""
  alerting:
    enabled: false
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:
    enabled: false
    # operator:
    #   resources: {}
    # webhook:
    #   resources: {}
  devops:
    enabled: false
    # resources: {}
    jenkinsMemoryLim: 2Gi
    jenkinsMemoryReq: 1500Mi
    jenkinsVolumeSize: 8Gi
    jenkinsJavaOpts_Xms: 1200m
    jenkinsJavaOpts_Xmx: 1600m
    jenkinsJavaOpts_MaxRAM: 2g
  events:
    enabled: false
    # operator:
    #   resources: {}
    # exporter:
    #   resources: {}
    # ruler:
    #   enabled: true
    #   replicas: 2
    #   resources: {}
  logging:
    enabled: false
    logsidecar:
      enabled: true
      replicas: 2
      # resources: {}
  metrics_server:
    enabled: false
  monitoring:
    storageClass: ""
    node_exporter:
      port: 9100
      # resources: {}
    # kube_rbac_proxy:
    #   resources: {}
    # kube_state_metrics:
    #   resources: {}
    # prometheus:
    #   replicas: 1
    #   volumeSize: 20Gi
    #   resources: {}
    #   operator:
    #     resources: {}
    # alertmanager:
    #   replicas: 1
    #   resources: {}
    # notification_manager:
    #   resources: {}
    #   operator:
    #     resources: {}
    #   proxy:
    #     resources: {}
    gpu:
      nvidia_dcgm_exporter:
        enabled: false
        # resources: {}
  multicluster:
    clusterRole: none
  network:
    networkpolicy:
      enabled: false
    ippool:
      type: none
    topology:
      type: none
  openpitrix:
    store:
      enabled: false
  servicemesh:
    enabled: false
    istio:
      components:
        ingressGateways:
        - name: istio-ingressgateway
          enabled: false
        cni:
          enabled: false
  edgeruntime:
    enabled: false
    kubeedge:
      enabled: false
      cloudCore:
        cloudHub:
          advertiseAddress:
            - ""
        service:
          cloudhubNodePort: "30000"
          cloudhubQuicNodePort: "30001"
          cloudhubHttpsNodePort: "30002"
          cloudstreamNodePort: "30003"
          tunnelNodePort: "30004"
        # resources: {}
        # hostNetWork: false
      iptables-manager:
        enabled: true
        mode: "external"
        # resources: {}
      # edgeService:
      #   resources: {}
  terminal:
    timeout: 600

```

## 2. 方式2：使用kubekey生成kubesphere.yaml

### 1. 准备脚本install-kk-config.sh
```shell
#!/bin/bash

export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.2 sh -
chmod 777 ./kk
echo "安装kubesphare运行时依赖..."
yum install -y openssl openssl-devel socat epel-release conntrack-tools 

./kk create config --with-kubernetes v1.20.4 --with-kubesphere v3.1.1

mv config-sample.yaml kubesphere.yaml

```
### 2. 执行脚本install-kk-config.sh
` sh install-kk-config.sh `


> 注意： 生成kubesphere.yaml后，要编辑此文件，将host属性部分改为自己集群的节点ip

# 3. 安装kubesphere集群

## 1. 如果没有kubekey，先安装kubekey

(1). 准备脚本install-kk.sh
```shell
#!/bin/bash

export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.2 sh -
chmod 777 ./kk
echo "安装kubesphare运行时依赖..."
yum install -y openssl openssl-devel socat epel-release conntrack-tools 
```
(2). 执行脚本install-kk.sh
` sh install-kk.sh `


## 2. 安装kubesphere

./kk create cluster -f kubesphere.yaml

## 3. 查看安装进度

```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f

```
