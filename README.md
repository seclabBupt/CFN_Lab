# CFN_Lab

## 一、环境搭建

### 1.软件安装

- docker安装

```shell
apt install docker.io
```

- k3d安装

```shell
vim install.sh
写入https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh的内容
chmod +x install.sh
./install.sh
```

- ovs安装

```shell
apt install openvswitch-switch
```

- 安装k3s

```shell
curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

### 2.开启onos控制器

```shell
docker pull onosproject/onos
docker run -p 8181:8181 --name onos-ui -d onosproject/onos:latest

ssh -p 8101 karaf@容器IP
app activate org.onosproject.openflow
app activate org.onosproject.fwd

Web页面：主机IP:8181/onos/ui
开启openflow、fwd服务
```

### 3.ovs拓扑搭建

```shell
ovs-vsctl add-br s1
ovs-vsctl set bridge s1 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14

ovs-vsctl add-br s2
ovs-vsctl set bridge s2 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14

ovs-vsctl add-br s3
ovs-vsctl set bridge s3 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14

ovs-vsctl add-br s4
ovs-vsctl set bridge s4 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14

ovs-vsctl add-br s5
ovs-vsctl set bridge s5 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14

ovs-vsctl add-br s6
ovs-vsctl set bridge s6 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14

ovs-vsctl add-port s1 s1s2p
ovs-vsctl add-port s1 s1s3p

ovs-vsctl add-port s2 s2s1p
ovs-vsctl add-port s2 s2s4p
ovs-vsctl add-port s2 s2s5p
ovs-vsctl add-port s2 s2s6p

ovs-vsctl add-port s3 s3s1p
ovs-vsctl add-port s3 s3s4p
ovs-vsctl add-port s3 s3s5p
ovs-vsctl add-port s3 s3s6p

ovs-vsctl add-port s4 s4s2p
ovs-vsctl add-port s4 s4s3p

ovs-vsctl add-port s5 s5s2p
ovs-vsctl add-port s5 s5s3p

ovs-vsctl add-port s6 s6s2p
ovs-vsctl add-port s6 s6s3p

ovs-vsctl set Interface s1s2p type=patch
ovs-vsctl set Interface s1s2p options:peer=s2s1p
ovs-vsctl set Interface s1s3p type=patch
ovs-vsctl set Interface s1s3p options:peer=s3s1p

ovs-vsctl set Interface s2s1p type=patch
ovs-vsctl set Interface s2s1p options:peer=s1s2p
ovs-vsctl set Interface s2s4p type=patch
ovs-vsctl set Interface s2s4p options:peer=s4s2p
ovs-vsctl set Interface s2s5p type=patch
ovs-vsctl set Interface s2s5p options:peer=s5s2p
ovs-vsctl set Interface s2s6p type=patch
ovs-vsctl set Interface s2s6p options:peer=s6s2p

ovs-vsctl set Interface s3s1p type=patch
ovs-vsctl set Interface s3s1p options:peer=s1s3p
ovs-vsctl set Interface s3s4p type=patch
ovs-vsctl set Interface s3s4p options:peer=s4s3p
ovs-vsctl set Interface s3s5p type=patch
ovs-vsctl set Interface s3s5p options:peer=s5s3p
ovs-vsctl set Interface s3s6p type=patch
ovs-vsctl set Interface s3s6p options:peer=s6s3p

ovs-vsctl set Interface s4s2p type=patch
ovs-vsctl set Interface s4s2p options:peer=s2s4p
ovs-vsctl set Interface s4s3p type=patch
ovs-vsctl set Interface s4s3p options:peer=s3s4p

ovs-vsctl set Interface s5s2p type=patch
ovs-vsctl set Interface s5s2p options:peer=s2s5p
ovs-vsctl set Interface s5s3p type=patch
ovs-vsctl set Interface s5s3p options:peer=s3s5p

ovs-vsctl set Interface s6s2p type=patch
ovs-vsctl set Interface s6s2p options:peer=s2s6p
ovs-vsctl set Interface s6s3p type=patch
ovs-vsctl set Interface s6s3p options:peer=s3s6p

ovs-vsctl set-controller s1 tcp:172.17.0.2:6653
ovs-vsctl set-controller s2 tcp:172.17.0.2:6653
ovs-vsctl set-controller s3 tcp:172.17.0.2:6653
ovs-vsctl set-controller s4 tcp:172.17.0.2:6653
ovs-vsctl set-controller s5 tcp:172.17.0.2:6653
ovs-vsctl set-controller s6 tcp:172.17.0.2:6653
                             （替换为onos容器IP）
```

### 4.集群搭建

```shell 
k3d cluster create c1 --registry-create c1-registry -s 1 -a 3
k3d cluster create c2 --registry-create c2-registry
k3d cluster create c3 --registry-create c3-registry

进入到每一个docker容器添加路由：
docker exec -it <容器ID> /bin/sh
route add -net 0.0.0.0 dev eth0（仓库为eth1）
exit

将集群docker网桥上的端口配置到ovs网桥上：
ip link set 端口 nomaster #删除网桥上端口
ovs-vsctl add-port 网桥 端口
```

 ### 5.添加用户

```shell
ip netns add h1
ip link add h1s4 type veth peer name s4h1
ip link set h1s4 netns h1
ip netns exec h1 ip addr add 10.0.4.11/24 dev h1s4
ip netns exec h1 ip link set h1s4 up
ip link set s4h1 up
ovs-vsctl add-port s4 s4h1

进入h1添加路由：
ip netns exec h1 bash
route add -net 0.0.0.0 dev h1s4
exit
```

### 6.使用kubectl命令

- 主机连接集群

```shell
ifconfig s1 up 
route add -net 172.27.0.0/16 dev s1
               （集群网段）
```

- 获取集群配置文件

```shell
k3d kubeconfig get c1（集群名称）

cd /etc/rancher/k3s
写入到k3s.yaml中
```

### 7. 集群镜像仓库使用

```shell
docker pull 镜像
docker tag 镜像 localhost:仓库端口/镜像
docker push localhost:仓库端口/镜像
```

```shell
部署deployment的yaml文件中：
image:集群仓库名称:5000/镜像
```

### 8.服务部署

- 制作镜像

```shell
docker run -itd --name ubuntu ubuntu:focal

apt-get install vim
apt-get install -y iproute2
apt-get install iputils-ping
apt install iperf
apt install tcpdump

docker commit ubuntu ubuntu20.04
docker tag ubuntu20.04 seclabdockerhub/sdnlabdockerhub:ubuntu20.04
docker login -u seclabdockerhub
docker push seclabdockerhub/sdnlabdockerhub:ubuntu20.04
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: ubuntu
  name: ubuntu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ubuntu
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ubuntu
    spec:
      containers:
      - image: c1-registry:5000/seclabdockerhub/sdnlabdockerhub:ubuntu20.04
        name: ubuntu
        resources: {}
        command: ["/bin/bash","-c","--"]
        args: ["while true; do sleep 30; done;"]
        securityContext:
          privileged: true
status: {}
```

- 内存、CPU

> http://t.zoukankan.com/wuyun-blog-p-9524292.html

- Iperf

> https://iperf.fr/iperf-doc.php

- k3s-service

```yaml
apiVersion: v1             
kind: Service             
metadata:
  name: ubuntu
  labels:
    app: ubuntu
spec:
  selector:                    #选择器，需要与pod 的命名是一致
    app: ubuntu
  type: NodePort      
  ports:
  - name:  ubuntu
    protocol: TCP
    port:  5001
    nodePort: 31600
    targetPort:  5001
```

```bash
pod内：
iperf -s -i 1 -w 1M
host内：
iperf -c 172.23.0.2 -i 1 -w 1M -p 31600
```

