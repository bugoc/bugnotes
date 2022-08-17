# Ubuntu基于kubeadm的Kubernetes 高可用集群部署

## 1. Kubernetes 集群说明

### 1.1 Kubernetes 组件

&emsp; &emsp;Kubernetes 集群集群[组件](https://kubernetes.io/zh-cn/docs/concepts/overview/components/)至少有一个主节点（Master）和一个工作节点（Node），下图是 Kubernetes 集群部署示意图。
![kubernetes_ha](kubernetes/Kuberentes_ha_20220810094641.png)

### 1.2 Kubernetes 安装列表

| 主机名 | IP 地址 | 角色 | 说明 |
| ------- | ------ | ----- | ----- |
| node01.wacos.io | 192.168.100.71 | master,node| Ubuntu、kubernetes、Keepalived、HAproxy |
| node02.wacos.io | 192.168.100.72 | master,node| Ubuntu、kubernetes、Keepalived、HAproxy |
| node03.wacos.io | 192.168.100.73 | master,node| Ubuntu、kubernetes、Keepalived、HAproxy |
| master_vip | 192.168.100.70 | VIP| |

```
sudo tee -a /etc/hosts <<EOF
192.168.100.71 node01.wacos.io node01
192.168.100.72 node02.wacos.io node02
192.168.100.73 node03.wacos.io node03
192.168.100.70 vip.wacos.io  
EOF
```

### 1.3 Kubernetes 网络规划

| 名称 | 地址 | 说明 |
| ---- | ---- | ----|
| 主机 | 192.168.100.0/24 | 主机网络 |
| Service | 10.96.0.0/16 | Service 地址 |
| Pod | 10.244.0.0/16 | Pod 地址|

## 2. Kubernetes 安装准备

### 2.1 关闭 Swap 分区

```
## 在线关闭
sudo swapoff -a

## 永久关闭
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
> 说明：编辑 kubelet 的配置文件 `/etc/sysconfig/kubelet`，设置其忽略 Swap 启用的状态错误。
> ```
> sudo tee -a /etc/sysconfig/kubelet <<EOF
> KUBELET_EXTRA_ARGS="--fail-swap-on=false"
> EOF
> ```

### 2.2 关闭 Selinux

```
## 在线关闭
sudo setenforce 0

## 永久关闭
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 2.3 [配置防火墙端口](https://kubernetes.io/zh-cn/docs/reference/ports-and-protocols/)

1. 控制平面
    ```
    ## Kubernetes API server
    sudo ufw allow 6443/tcp
    sudo ufw allow 8443/tcp

    ## etcd server client API
    sudo ufw allow 2379:2381/tcp

    ## Kubelet API
    sudo ufw allow 10250/tcp

    ## kube-scheduler
    sudo ufw allow 10259/tcp

    
    ## kube-controller-manager
    sudo ufw allow 10257/tcp
    sudo ufw allow 443/tcp

    # kube-proxy
    sudo ufw allow 10256/tcp

    ## kube-dns
    sudo ufw allow 53/udp
    sudo ufw allow 53/tcp
    sudo ufw allow 9153/tcp
    
    ## kube-network
    sudo ufw allow 8472/udp
    sudo ufw allow 4789/udp
    sudo ufw allow 5473/tcp
    sudo ufw allow 179/tcp
    sudo ufw allow 9098:9099/tcp

    sudo ufw allow from 192.168.100.0/24
    sudo ufw allow from 10.96.0.0/16
    sudo ufw allow from 10.244.0.0/16
    sudo ufw reload
    ```

2. 工作节点
    ```
    ## Kubelet API
    sudo ufw allow 10250/tcp

    ## NodePort Services
    sudo ufw allow 30000:32767/tcp
    sudo ufw allow 443/tcp

    # kube-proxy
    sudo ufw allow 10256/tcp

    ## kube-dns
    sudo ufw allow 53/udp
    sudo ufw allow 53/tcp
    sudo ufw allow 9153/tcp

    ## kube-network
    sudo ufw allow 8472/udp
    sudo ufw allow 4789/udp
    sudo ufw allow 5473/tcp
    sudo ufw allow 179/tcp    
    sudo ufw allow 9098:9099/tcp

    sudo ufw allow from 192.168.100.0/24
    sudo ufw allow from 10.96.0.0/16
    sudo ufw allow from 10.244.0.0/16
    sudo ufw reload
    ```
   > ufw 命令参考：https://linuxconfig.org/how-to-install-and-use-ufw-firewall-on-linux

### 2.4 允许 iptables 检查桥接流量

```
sudo modprobe br_netfilter
sudo tee /etc/modules-load.d/k8s.conf <<EOF
br_netfilter
EOF
sudo lsmod|grep br_netfilter

sudo tee /etc/sysctl.d/k8s.conf <<EOF  
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
sudo sysctl -p
sudo sysctl -a |grep bridge
```

### 2.5 [设置IPVS模块](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md)

```
sudo tee /etc/modules-load.d/ipvs.modules  <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

sudo modprobe -- ip_vs
sudo modprobe -- ip_vs_rr
sudo modprobe -- ip_vs_wrr
sudo modprobe -- ip_vs_sh
sudo modprobe -- nf_conntrack
sudo lsmod | grep -E "ip_vs|nf_conntrack"
```

## 3. 容器运行时部署

&emsp; &emsp;容器运行时可以用使用 Docker 或者 Containerd，两者选其一即可。

### 3.1 [Docker](https://docs.docker.com/engine/install/ubuntu/)

1. 设置 Docker 仓库
   ```
   # update apt 
   sudo apt-get update
   sudo apt-get install \
     ca-certificates \
     curl \
     gnupg \
     lsb-release

   # Add Docker’s official GPG key
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg  

   echo \
   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

2. 安装 Docker
   ```
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
   ```

3. 配置Docker
   ```
   sudo usermod -G docker $USER

   ## Create /etc/docker directory.
   sudo mkdir -p /etc/docker
   
   ## Setup daemon.
   sudo tee /etc/docker/daemon.json <<EOF
   {   
     "registry-mirrors": [
          "https://docker.mirrors.ustc.edu.cn",
          "https://reg-mirror.qiniu.com",
         "https://dockerhub.azk8s.cn",
         "https://mirror.ccs.tencentyun.com"
       ],
     "data-root": "/opt/docker", 
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {  
       "max-size": "10m",
       "max-file": "5"
   
     },
     "storage-driver": "overlay2",
     "storage-opts": [
       "overlay2.override_kernel_check=true"
     ]
   }
   EOF   
   ```

4. 重启 Docker
   ```
   sudo mkdir -p /etc/systemd/system/docker.service.d

   # Restart Docker
   sudo systemctl daemon-reload
   sudo systemctl restart docker

   # 切换到 docker 组，普通用户可执行docker命令
   newgrp docker
   docker info
   ```

### 3.2 [Containerd](https://containerd.io/)

1. [下载和安装Containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)。
    ```
    wget https://github.com/containerd/containerd/releases/download/v1.6.6/containerd-1.6.6-linux-amd64.tar.gz
    sudo tar Cxzvf /usr/local containerd-1.6.6-linux-amd64.tar.gz
    ```

2. 添加 systemctl 启动程序：/usr/lib/systemd/system/containerd.service。
    ```
    sudo tee /usr/lib/systemd/system/containerd.service <<EOF
    # Copyright The containerd Authors.
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    
    [Unit]
    Description=containerd container runtime
    Documentation=https://containerd.io
    After=network.target local-fs.target
    
    [Service]
    #uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
    #Environment="ENABLE_CRI_SANDBOXES=sandboxed"
    ExecStartPre=-/sbin/modprobe overlay
    ExecStart=/usr/local/bin/containerd
    
    Type=notify
    Delegate=yes
    KillMode=process
    Restart=always
    RestartSec=5
    # Having non-zero Limit*s causes performance problems due to accounting overhead
    # in the kernel. We recommend using cgroups to do container-local accounting.
    LimitNPROC=infinity
    LimitCORE=infinity
    LimitNOFILE=infinity
    # Comment TasksMax if your systemd version does not supports it.
    # Only systemd 226 and above support this version.
    TasksMax=infinity
    OOMScoreAdjust=-999
    
    [Install]
    WantedBy=multi-user.target   
    EOF
 
    sudo systemctl daemon-reload
    sudo systemctl enable --now containerd
    ```

3. 创建containerd配置文件。
    ```
    sudo mkdir -p /etc/containerd
    containerd config default |sudo tee /etc/containerd/config.toml
    ```

4. 修改containerd配置。
    ```
    sudo sed -i 's#root = "/var/lib/containerd"#root = "/opt/containerd"#g' /etc/containerd/config.toml
    sudo sed -i 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' /etc/containerd/config.toml
    sudo sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
    ```

5. 修改kubelet配置。
    ```
    sudo sed -i '/Service/a\Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroup-driver=systemd"' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    ```

6. 配置crictl客户端配置。
    ```
    sudo tee /etc/crictl.yaml <<EOF
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
    timeout: 10
    debug: false
    EOF
    ```

## 4. kubernetes部署

### 4.1 配置VIP

#### 4.1.1 [安装docker-compose](https://docs.docker.com/compose/install/compose-plugin/#installing-compose-on-linux-systems)

&emsp; &emsp;使用 docker compose 临时启动 haproxy 和 nginx，确保 VIP 可用。

1. 下载 docker-compose
    ```
    sudo curl -SL https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
    ```

2. docker-compose 增加执行权限
    ```
    sudo chmod a+x /usr/local/bin/docker-compose
    ```

3. 验证 docker-compose
    ```
    docker-compose version
    ```

#### 4.1.2 编写 keepalived 配置

&emsp; &emsp;在 `/etc/keepalived` 目录下新增 `keepalived.conf` 配置文件和 `check_apiserver.sh` 探测脚本。

1. `keepalived.conf`
    ```
    sudo mkdir -p /etc/keepalived
 
    sudo tee /etc/keepalived/keepalived.conf <<EOF
    ! Configuration File for keepalived
    global_defs {
       router_id LVS_DEVEL
    }
    vrrp_script check_apiserver {
       script "/etc/keepalived/check_apiserver.sh"
       interval 3
       weight -2
       fall 10
       rise 2
    }
    
    vrrp_instance VI_1 {
       state MASTER
       interface ens33
       virtual_router_id 59
       priority 101
       authentication {
             auth_type PASS
             auth_pass 1111
       }
       virtual_ipaddress {
             192.168.100.70
       }
       track_script {
             check_apiserver
       }
    }
    EOF
    ```
 
2. `check_apiserver.sh` 
    ```
    sudo tee /etc/keepalived/check_apiserver.sh <<'EOF'
    #!/bin/bash
    
    APISERVER_VIP="192.168.100.70"
    APISERVER_DEST_PORT=8443
    
    errorExit() {
       echo "*** $*" 1>&2
       exit 1
    }
    
    curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
    if ip addr | grep -q ${APISERVER_VIP}; then
       curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:$ {APISERVER_DEST_PORT}/"
    fi
    EOF
 
    sudo chmod u+x /etc/keepalived/check_apiserver.sh
    ``` 

#### 4.1.3 编写 haproxy 配置

- 在 `/etc/haproxy` 目录下新增 `haproxy.cfg` 配置文件。
   ```
   sudo mkdir -p /etc/haproxy
   
   sudo tee /etc/haproxy/haproxy.cfg <<EOF
   #---------------------------------------------------------------------
   # Global settings
   #---------------------------------------------------------------------
   global
       log /dev/log local0
       log /dev/log local1 notice
       daemon
   
   #---------------------------------------------------------------------
   # common defaults that all the 'listen' and 'backend' sections will
   # use if not designated in their block
   #---------------------------------------------------------------------
   defaults
       mode                    http
       log                     global
       option                  httplog
       option                  dontlognull
       option http-server-close
       option forwardfor       except 127.0.0.0/8
       option                  redispatch
       retries                 1
       timeout http-request    10s
       timeout queue           20s
       timeout connect         5s
       timeout client          20s
       timeout server          20s
       timeout http-keep-alive 10s
       timeout check           10s
   
   #---------------------------------------------------------------------
   # apiserver frontend which proxys to the control plane nodes
   #---------------------------------------------------------------------
   frontend apiserver
       bind *:8443
       mode tcp
       option tcplog
       default_backend apiserver
   
   #---------------------------------------------------------------------
   # round robin balancing for apiserver
   #---------------------------------------------------------------------
   backend apiserver
       option httpchk GET /healthz
       http-check expect status 200
       mode tcp
       option ssl-hello-chk
       balance     roundrobin
           server srv1 192.168.100.71:6443 check
           server srv2 192.168.100.72:6443 check
           server srv3 192.168.100.73:6443 check
   EOF
   ```

#### 4.1.4 docker-compose 启动 keepalived 和 haproxy

1. 配置 docker-compose.yaml
    ```
    tee docker-compose.yaml <<EOF
    version: '2'
    services:
      keepalived:
        image: osixia/keepalived:2.0.20
        container_name: keepalived
        hostname: keepalived
        volumes:
        - /etc/keepalived/check_apiserver.sh:/container/service/keepalived/assets/check_apiserver.sh
        - /etc/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf
        cap_add:
          - NET_ADMIN
        command: --copy-service
        privileged: true
        network_mode: host
        restart: always
      haproxy:
        image: haproxy:2.1.4
        container_name: haproxy
        restart: always
        network_mode: host  
        ports:
        - 8443:8443
        volumes:
        - /etc/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
        environment:
          TIME_ZONE: Asia/Shanghai	
    EOF    
    ``` 

2. 启动服务
    ```
    docker-compose up -d 
    ```

### 4.2 [安装 kubeadm、kubelet 和 kubectl](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) 

1. 更新 apt 索引和安装 kubernetes 仓库所需的依赖包。
    ```
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    ```

2. 添加签名证书和 kubernetes 仓库。
    ```
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
 
    sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
    deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
    EOF
   ```

3. 更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl。
    ```
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
 
    # 安装指定版本
    sudo apt-get install -y kubelet=1.22.12-00 kubeadm=1.22.12-00 kubectl=1.22.12-00
    # 防止自动更新
    sudo apt-mark hold kubelet kubeadm kubectl
   ```

### 4.3 拉取镜像文件

&emsp; &emsp;由于众所周知的原因，无法直接拉取谷歌站点镜像，需要事先准备。

1. 列出 kubernetes 镜像信息。
    ```
    $ kubeadm config images list
    k8s.gcr.io/kube-apiserver:v1.24.3
    k8s.gcr.io/kube-controller-manager:v1.24.3
    k8s.gcr.io/kube-scheduler:v1.24.3
    k8s.gcr.io/kube-proxy:v1.24.3
    k8s.gcr.io/pause:3.7
    k8s.gcr.io/etcd:3.5.3-0
    k8s.gcr.io/coredns/coredns:v1.8.6
    ```
    > 下载镜像：`kubeadm config images pull --config kubeadm-config.yaml`

2. 编辑脚本拉取镜像
    ```
    tee pull_k8s_images.sh <<'EOF'
    #!/bin/bash
    #
    # FileName: pull_k8s_gcr.sh
    # DESC: pull k8s images from aliyun
    
    KUBE_VERSION="v1.22.12"
    KUBE_PAUSE_VERSION="3.7"
    ETCD_VERSION="3.5.3-0"
    CORE_DNS_VERSION="1.8.6"
    
    GCR_URL="k8s.gcr.io"
    ALIYUN_URL="registry.cn-hangzhou.aliyuncs.com/google_containers"
    
    images=(kube-proxy:${KUBE_VERSION}
    kube-scheduler:${KUBE_VERSION}
    kube-controller-manager:${KUBE_VERSION}
    kube-apiserver:${KUBE_VERSION}
    pause:${KUBE_PAUSE_VERSION}
    etcd:${ETCD_VERSION}
    coredns:${CORE_DNS_VERSION})
    
    for imageName in ${images[@]} ; do
      sudo docker pull $ALIYUN_URL/$imageName
      sudo docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
      sudo docker rmi $ALIYUN_URL/$imageName
    done
    EOF
    ```

3. 拉取镜像
    ```
    chmod u+x pull_k8s_images.sh
    ./pull_k8s_images.sh
    ```

### 4.4 安装第一个控制节点

&emsp; &emsp;haproxy 和 keepalived Pod通过静态方式安装上，并使用 haproxy 的 VIP 作为 API Server 的服务地址。

#### 4.4.1 生成并配置 kubeadm_config.yaml 

1. 生成 kubeadm_config.yaml（可选）
    ```
    kubeadm config print init-defaults  > kubeadm-config.yaml
    ```

2. 配置 kubeadm_config.yaml
    ```
    tee kubeadm_config.yaml <<EOF
    apiVersion: kubeadm.k8s.io/v1beta3
    bootstrapTokens:
    - groups:
      - system:bootstrappers:kubeadm:default-node-token
      token: abcdef.0123456789abcdef
      ttl: 24h0m0s
      usages:
      - signing
      - authentication
    kind: InitConfiguration
    localAPIEndpoint:
      advertiseAddress: 192.168.100.71
      bindPort: 6443
    nodeRegistration:
      criSocket: /var/run/dockershim.sock
      imagePullPolicy: IfNotPresent
      name: node01
      taints: null
    ---
    apiServer:
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /opt/etcd
    imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    controlPlaneEndpoint: "vip.wacos.io:8443"
    kubernetesVersion: 1.22.12
    networking:
      dnsDomain: cluster.local
      serviceSubnet: 10.96.0.0/12
      podSubnet: 10.244.0.0/16
    ---
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    cgroupDriver: systemd
    ---
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: ipvs
    EOF
    ```

#### 4.4.2 初始化集群

- 执行以下命令执行初始化。
   ```
   sudo kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.log
   ```

- Kubernetes 不支持 Swap，安装前需要关闭，如果依然启用，可通过以下命令忽略错误。
   ```
   sudo kubeadm init --config=kubeadm-config.yaml --upload-certs --ignore-preflight-errors=Swap| tee kubeadm-init.log
   ```

- 执行成功，有如下提示信息。
   ```
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

   You can now join any number of the control-plane node running the following command on each as root:

   kubeadm join vip.wacos.io:8443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:690bc30c93d4ab231484aaeff03b7b29fa055dc91c78f7d3727834b6b9f07581 \
      --control-plane --certificate-key ce8c57a196a6b9024e046cd497cdd09525cf38b9a8449b71367ecc9cb304429f

   Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
   As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
   "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

   Then you can join any number of worker nodes by running the following on each as root:

   kubeadm join vip.wacos.io:8443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:690bc30c93d4ab231484aaeff03b7b29fa055dc91c78f7d3727834b6b9f07581
   ```

- 配置 kubectl 客户端，查看集群信息
   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config

   kubectl cluster-info

   kubectl get node
   NAME    STATUS     ROLES                  AGE     VERSION
   node01  NotReady   control-plane,master   4m10s   v1.22.12
   ```

#### 4.4.3 [安装网络插件](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)

1. 下载 Tigera Calico operator 并安装。
    ```
    curl -O https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
    
    kubectl apply -f tigera-operator.yaml
    ```

2. 下载安装配置，并修改对应的 Pod 地址池，`kubectl -n kube-system describe configmap kubeadm-config` 。
    ```
    curl -O https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml

    kubectl apply -f custom-resources.yaml
    ```
    > kubectl taint nodes --all node-role.kubernetes.io/master-

3. 查看运行状态，直至所有 Pod 变为 Running 状态。
    ```
    watch kubectl get pods -n calico-system
    ```

4. 安装 [calicoctl](https://projectcalico.docs.tigera.io/maintenance/clis/calicoctl/install) 客户端。
    ```
    sudo curl -L https://github.com/projectcalico/calico/releases/download/v3.23.3/calicoctl-linux-amd64 -o /usr/bin/calicoctl

    sudo chmod a+x /usr/bin/calicoctl
    sudo ln -s /usr/bin/calicoctl /usr/bin/kubectl-calico 
    ```    

5. 配置[calicoctl](https://projectcalico.docs.tigera.io/maintenance/clis/calicoctl/configure/) 访问集群。
    ```
    sudo mkdir -p /etc/calico
    sudo tee /etc/calico/calicoctl.cfg <<EOF
    apiVersion: projectcalico.org/v3
    kind: CalicoAPIConfig
    metadata:
    spec:
     datastoreType: "kubernetes"
     kubeconfig: "~/.kube/config"
    EOF
    ```
6. 执行命令查看
    ```
    sudo calicoctl node status
    ```

### 4.5 其他节点加入集群

1. 控制节点加入集群
   ```
   sudo kubeadm join vip.wacos.io:8443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:690bc30c93d4ab231484aaeff03b7b29fa055dc91c78f7d3727834b6b9f07581 \
      --control-plane --certificate-key ce8c57a196a6b9024e046cd497cdd09525cf38b9a8449b71367ecc9cb304429f   
   ```

2. Node 节点加入集群
   ```
   sudo kubeadm join vip.wacos.io:8443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:690bc30c93d4ab231484aaeff03b7b29fa055dc91c78f7d3727834b6b9f07581
   ```

### 4.6 将 VIP 切换到 Kubernetes 静态 Pod

1. 去除 master 节点污点，允许运行 Pod。
    ```
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

2. 在所有 master 节点`/etc/kubernetes/manifests/` 目录下创建 `keepalived.yaml` 和 `haproxy.yaml`   静态 Pod 配置文件。

   - `keepalived.yaml`
      ```
      sudo tee /etc/kubernetes/manifests/keepalived.yaml <<EOF
      apiVersion: v1
      kind: Pod
      metadata:
        annotations:
          scheduler.alpha.kubernetes.io/critical-pod: ""
        name: keepalived
        namespace: kube-system
      spec:
        volumes:
        - name: keepalived-conf
          hostPath:
            path: /etc/keepalived/
        containers:
        - name: keepalived
          image: osixia/keepalived:2.0.20
          imagePullPolicy: IfNotPresent
          volumeMounts:
          - name: keepalived-conf
            mountPath: /container/service/keepalived/assets/keepalived.conf
            subPath: keepalived.conf
            readOnly: true
          - name: keepalived-conf
            mountPath: /etc/keepalived/check_apiserver.sh
            subPath: check_apiserver.sh
            readOnly: true
          securityContext:
            capabilities:
              add:
              - NET_ADMIN
          args: 
          - --copy-service
        hostNetwork: true
        priorityClassName: system-cluster-critical
      EOF

      sudo chmod 600  /etc/kubernetes/manifests/keepalived.yaml 
      ```

   - `haproxy.yaml`
      ```
      sudo tee sudo tee /etc/kubernetes/manifests/haproxy.yaml <<EOF
      apiVersion: v1
      kind: Pod
      metadata:
        annotations:
          scheduler.alpha.kubernetes.io/critical-pod: ""
        name: haproxy
        namespace: kube-system
      spec:
        volumes:
        - name: haproxy-conf
          hostPath:
            type: File
            path: /etc/haproxy/haproxy.cfg
        containers:
        - name: haproxy
          image: haproxy:2.1.4
          imagePullPolicy: IfNotPresent
          ports:
          - name: apiserver
            containerPort: 8443
            hostPort: 8443
          livenessProbe:
            failureThreshold: 8
            tcpSocket:
              port: 8443
            periodSeconds: 10  
            initialDelaySeconds: 15
            timeoutSeconds: 15
          volumeMounts:
          - name: haproxy-conf
            mountPath: /usr/local/etc/haproxy/haproxy.cfg
            readOnly: true
        hostNetwork: true    
        priorityClassName: system-cluster-critical
      EOF

      sudo chmod 600  /etc/kubernetes/manifests/haproxy.yaml 
      ```

3. 检查所有 haproxy 和 keepalived 的 Pod 都处于 Running 状态。关闭  kubelet ，在关闭 docker-compose，最后启动 kubelet。
    ```
    kubectl get pods -n kube-system

    sudo systemctl stop kubelet 

    docker-compose down -v

    sudo systemctl restart kubelet
    ```

4. 检查所有 Pod 状态
    ```
    kubectl get pods -A
    ```

5. 恢复污点
    ```
    kubectl taint nodes --all node-role.kubernetes.io/master=:NoSchedule
    ```

### 4.7 安装 metrics-server 组件

&emsp; &emsp; [metrics-server](https://github.com/kubernetes-sigs/metrics-server/) 是 Cluster 的核心监控数据的聚合器，kubeadm 默认是没有部署的；是一个扩展的 APIServer，依赖于 API Aggregator。所以，在安装 Metrics Server 之前需要先在 kube-apiserver 中开启 [API Aggregator](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/)。

1. 检查 API Server 是否开启Aggregator Routing，查看 API Server 是否具有 `--enable-aggregator-routing=true` 选项。
    ```
    ps -ef |grep apiserver |grep -v grep |grep enable-aggregator-routing
    ```

2. 修改 API Server 配置文件 `/etc/kubernetes/manifests/kube-apiserver.yaml`，增加 `--enable-aggregator-routing=true` 选项。
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.100.71:6443
      creationTimestamp: null
      labels:
        component: kube-apiserver
        tier: control-plane
      name: kube-apiserver
      namespace: kube-system
    spec:
      containers:
      - command:
        - kube-apiserver
        - --advertise-address=192.168.100.71
        - --allow-privileged=true
        - --authorization-mode=Node,RBAC
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --enable-admission-plugins=NodeRestriction
        - --enable-bootstrap-token-auth=true
        - --enable-aggregator-routing=true # 新增
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        - --etcd-servers=https://127.0.0.1:2379
        - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
        - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
        - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
        - --requestheader-allowed-names=front-proxy-client
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-username-headers=X-Remote-User
        - --secure-port=6443
        - --service-account-issuer=https://kubernetes.default.svc.cluster.local
        - --service-account-key-file=/etc/kubernetes/pki/sa.pub
        - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
        - --service-cluster-ip-range=10.96.0.0/12
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.12
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 8
          httpGet:
            host: 192.168.100.71
            path: /livez
            port: 6443
            scheme: HTTPS
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 15
        name: kube-apiserver
        readinessProbe:
          failureThreshold: 3
          httpGet:
            host: 192.168.100.71
            path: /readyz
            port: 6443
            scheme: HTTPS
          periodSeconds: 1
          timeoutSeconds: 15
        resources:
          requests:
            cpu: 250m
        startupProbe:
          failureThreshold: 24
          httpGet:
            host: 192.168.100.71
            path: /livez
            port: 6443
            scheme: HTTPS
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 15
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certs
          readOnly: true
        - mountPath: /etc/ca-certificates
          name: etc-ca-certificates
          readOnly: true
        - mountPath: /etc/pki
          name: etc-pki
          readOnly: true
        - mountPath: /etc/kubernetes/pki
          name: k8s-certs
          readOnly: true
        - mountPath: /usr/local/share/ca-certificates
          name: usr-local-share-ca-certificates
          readOnly: true
        - mountPath: /usr/share/ca-certificates
          name: usr-share-ca-certificates
          readOnly: true
      hostNetwork: true
      priorityClassName: system-node-critical
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      volumes:
      - hostPath:
          path: /etc/ssl/certs
          type: DirectoryOrCreate
        name: ca-certs
      - hostPath:
          path: /etc/ca-certificates
          type: DirectoryOrCreate
        name: etc-ca-certificates
      - hostPath:
          path: /etc/pki
          type: DirectoryOrCreate
        name: etc-pki
      - hostPath:
          path: /etc/kubernetes/pki
          type: DirectoryOrCreate
        name: k8s-certs
      - hostPath:
          path: /usr/local/share/ca-certificates
          type: DirectoryOrCreate
        name: usr-local-share-ca-certificates
      - hostPath:
          path: /usr/share/ca-certificates
          type: DirectoryOrCreate
        name: usr-share-ca-certificates
    status: {}
    ```

3. 下载部署 yaml 文件。
    ```
    wget -c https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.1/components.yaml
    ```

4. 修改配置文件。
    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      labels:
        k8s-app: metrics-server
      name: metrics-server
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      labels:
        k8s-app: metrics-server
        rbac.authorization.k8s.io/aggregate-to-admin: "true"
        rbac.authorization.k8s.io/aggregate-to-edit: "true"
        rbac.authorization.k8s.io/aggregate-to-view: "true"
      name: system:aggregated-metrics-reader
    rules:
    - apiGroups:
      - metrics.k8s.io
      resources:
      - pods
      - nodes
      verbs:
      - get
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      labels:
        k8s-app: metrics-server
      name: system:metrics-server
    rules:
    - apiGroups:
      - ""
      resources:
      - nodes/metrics
      verbs:
      - get
    - apiGroups:
      - ""
      resources:
      - pods
      - nodes
      verbs:
      - get
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      labels:
        k8s-app: metrics-server
      name: metrics-server-auth-reader
      namespace: kube-system
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: extension-apiserver-authentication-reader
    subjects:
    - kind: ServiceAccount
      name: metrics-server
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      labels:
        k8s-app: metrics-server
      name: metrics-server:system:auth-delegator
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:auth-delegator
    subjects:
    - kind: ServiceAccount
      name: metrics-server
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      labels:
        k8s-app: metrics-server
      name: system:metrics-server
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:metrics-server
    subjects:
    - kind: ServiceAccount
      name: metrics-server
      namespace: kube-system
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        k8s-app: metrics-server
      name: metrics-server
      namespace: kube-system
    spec:
      ports:
      - name: https
        port: 443
        protocol: TCP
        targetPort: https
      selector:
        k8s-app: metrics-server
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        k8s-app: metrics-server
      name: metrics-server
      namespace: kube-system
    spec:
      selector:
        matchLabels:
          k8s-app: metrics-server
      strategy:
        rollingUpdate:
          maxUnavailable: 0
      template:
        metadata:
          labels:
            k8s-app: metrics-server
        spec:
          containers:
          - args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --kubelet-preferred-address-types=InternalIP # 删除ExternalIP,Hostname
            - --kubelet-use-node-status-port
            - --metric-resolution=15s
            - --kubelet-insecure-tls  # 新增
            image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.1 # 修改地址
            imagePullPolicy: IfNotPresent
            livenessProbe:
              failureThreshold: 3
              httpGet:
                path: /livez
                port: https
                scheme: HTTPS
              periodSeconds: 10
            name: metrics-server
            ports:
            - containerPort: 4443
              name: https
              protocol: TCP
            readinessProbe:
              failureThreshold: 3
              httpGet:
                path: /readyz
                port: https
                scheme: HTTPS
              initialDelaySeconds: 20
              periodSeconds: 10
            resources:
              requests:
                cpu: 100m
                memory: 200Mi
            securityContext:
              allowPrivilegeEscalation: false
              readOnlyRootFilesystem: true
              runAsNonRoot: true
              runAsUser: 1000
            volumeMounts:
            - mountPath: /tmp
              name: tmp-dir
          nodeSelector:
            kubernetes.io/os: linux
          priorityClassName: system-cluster-critical
          serviceAccountName: metrics-server
          volumes:
          - emptyDir: {}
            name: tmp-dir
    ---
    apiVersion: apiregistration.k8s.io/v1
    kind: APIService
    metadata:
      labels:
        k8s-app: metrics-server
      name: v1beta1.metrics.k8s.io
    spec:
      group: metrics.k8s.io
      groupPriorityMinimum: 100
      insecureSkipTLSVerify: true
      service:
        name: metrics-server
        namespace: kube-system
      version: v1beta1
      versionPriority: 100
    ```

5. 检查 metrics-server 运行状态
    ```
    kubectl -n kube-system get pod -l k8s-app=metrics-server

    kubectl describe svc metrics-server -n kube-system
    ```

6. 执行检查命令
    ```
    $ kubectl top node
    NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
    node01   221m         11%    1818Mi          63%       
    node02   206m         10%    1854Mi          65%       
    node03   233m         11%    1738Mi          61%       

    $ kubectl top pod -n kube-system
    NAME                              CPU(cores)   MEMORY(bytes)   
    coredns-7d89d9b6b8-s5s96          2m           16Mi            
    coredns-7d89d9b6b8-tgq8w          2m           13Mi            
    etcd-node01                       41m          81Mi            
    etcd-node02                       36m          84Mi            
    etcd-node03                       53m          80Mi            
    haproxy-node01                    3m           66Mi            
    haproxy-node02                    2m           68Mi            
    haproxy-node03                    2m           67Mi            
    keepalived-node01                 17m          10Mi            
    keepalived-node02                 11m          15Mi            
    keepalived-node03                 10m          16Mi            
    kube-apiserver-node01             71m          279Mi           
    kube-apiserver-node02             65m          275Mi           
    kube-apiserver-node03             61m          286Mi           
    kube-controller-manager-node01    3m           17Mi            
    kube-controller-manager-node02    16m          51Mi            
    kube-controller-manager-node03    2m           15Mi            
    kube-proxy-2scvg                  10m          19Mi            
    kube-proxy-jrpdm                  10m          17Mi            
    kube-proxy-v6s44                  5m           22Mi            
    kube-scheduler-node01             3m           18Mi            
    kube-scheduler-node02             3m           14Mi            
    kube-scheduler-node03             3m           16Mi            
    metrics-server-698478bb6c-djqkr   4m           19Mi      
    ```

### 4.8 安装kubernetes-dashboard组件

1. 下载 [kubernetes-dashboard](https://github.com/kubernetes/dashboard) 部署 yaml 文件。
    ```
    wget -c https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
    ```

2. 在 yaml 文件修改暴露端口方式为NodePort和添加 RBAC 权限。
    ```
    # Copyright 2017 The Kubernetes Authors.
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    
    apiVersion: v1
    kind: Namespace
    metadata:
      name: kubernetes-dashboard
    
    ---
    
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard
      namespace: kubernetes-dashboard
    
    ---
    
    kind: Service
    apiVersion: v1
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard
      namespace: kubernetes-dashboard
    spec:
      type: NodePort #新增
      ports:
        - port: 443
          targetPort: 8443
          nodePort: 30000 #新增
      selector:
        k8s-app: kubernetes-dashboard
    
    ---
    
    apiVersion: v1
    kind: Secret
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard-certs
      namespace: kubernetes-dashboard
    type: Opaque
    
    ---
    
    apiVersion: v1
    kind: Secret
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard-csrf
      namespace: kubernetes-dashboard
    type: Opaque
    data:
      csrf: ""
    
    ---
    
    apiVersion: v1
    kind: Secret
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard-key-holder
      namespace: kubernetes-dashboard
    type: Opaque
    
    ---
    
    kind: ConfigMap
    apiVersion: v1
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard-settings
      namespace: kubernetes-dashboard
    
    ---
    
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard
      namespace: kubernetes-dashboard
    rules:
      # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
      - apiGroups: [""]
        resources: ["secrets"]
        resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
        verbs: ["get", "update", "delete"]
        # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
      - apiGroups: [""]
        resources: ["configmaps"]
        resourceNames: ["kubernetes-dashboard-settings"]
        verbs: ["get", "update"]
        # Allow Dashboard to get metrics.
      - apiGroups: [""]
        resources: ["services"]
        resourceNames: ["heapster", "dashboard-metrics-scraper"]
        verbs: ["proxy"]
      - apiGroups: [""]
        resources: ["services/proxy"]
        resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
        verbs: ["get"]
    
    ---
    
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard
    rules:
      # Allow Metrics Scraper to get metrics from the Metrics server
      - apiGroups: ["metrics.k8s.io"]
        resources: ["pods", "nodes"]
        verbs: ["get", "list", "watch"]
    
    ---
    
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard
      namespace: kubernetes-dashboard
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: kubernetes-dashboard
    subjects:
      - kind: ServiceAccount
        name: kubernetes-dashboard
        namespace: kubernetes-dashboard
    
    ---
    
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kubernetes-dashboard
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kubernetes-dashboard
    subjects:
      - kind: ServiceAccount
        name: kubernetes-dashboard
        namespace: kubernetes-dashboard
    
    ---
    
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      name: kubernetes-dashboard
      namespace: kubernetes-dashboard
    spec:
      replicas: 1
      revisionHistoryLimit: 10
      selector:
        matchLabels:
          k8s-app: kubernetes-dashboard
      template:
        metadata:
          labels:
            k8s-app: kubernetes-dashboard
        spec:
          securityContext:
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: kubernetes-dashboard
              image: kubernetesui/dashboard:v2.6.0
              imagePullPolicy: Always
              ports:
                - containerPort: 8443
                  protocol: TCP
              args:
                - --auto-generate-certificates
                - --namespace=kubernetes-dashboard
                # Uncomment the following line to manually specify Kubernetes API server Host
                # If not specified, Dashboard will attempt to auto discover the API server and connect
                # to it. Uncomment only if the default does not work.
                # - --apiserver-host=http://my-address:port
              volumeMounts:
                - name: kubernetes-dashboard-certs
                  mountPath: /certs
                  # Create on-disk volume to store exec logs
                - mountPath: /tmp
                  name: tmp-volume
              livenessProbe:
                httpGet:
                  scheme: HTTPS
                  path: /
                  port: 8443
                initialDelaySeconds: 30
                timeoutSeconds: 30
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                runAsUser: 1001
                runAsGroup: 2001
          volumes:
            - name: kubernetes-dashboard-certs
              secret:
                secretName: kubernetes-dashboard-certs
            - name: tmp-volume
              emptyDir: {}
          serviceAccountName: kubernetes-dashboard
          nodeSelector:
            "kubernetes.io/os": linux
          # Comment the following tolerations if Dashboard must not be deployed on master
          tolerations:
            - key: node-role.kubernetes.io/master
              effect: NoSchedule
    
    ---
    
    kind: Service
    apiVersion: v1
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      name: dashboard-metrics-scraper
      namespace: kubernetes-dashboard
    spec:
      ports:
        - port: 8000
          targetPort: 8000
      selector:
        k8s-app: dashboard-metrics-scraper
    
    ---
    
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      name: dashboard-metrics-scraper
      namespace: kubernetes-dashboard
    spec:
      replicas: 1
      revisionHistoryLimit: 10
      selector:
        matchLabels:
          k8s-app: dashboard-metrics-scraper
      template:
        metadata:
          labels:
            k8s-app: dashboard-metrics-scraper
        spec:
          securityContext:
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: dashboard-metrics-scraper
              image: kubernetesui/metrics-scraper:v1.0.8
              ports:
                - containerPort: 8000
                  protocol: TCP
              livenessProbe:
                httpGet:
                  scheme: HTTP
                  path: /
                  port: 8000
                initialDelaySeconds: 30
                timeoutSeconds: 30
              volumeMounts:
              - mountPath: /tmp
                name: tmp-volume
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                runAsUser: 1001
                runAsGroup: 2001
          serviceAccountName: kubernetes-dashboard
          nodeSelector:
            "kubernetes.io/os": linux
          # Comment the following tolerations if Dashboard must not be deployed on master
          tolerations:
            - key: node-role.kubernetes.io/master
              effect: NoSchedule
          volumes:
            - name: tmp-volume
              emptyDir: {}
    
    ######################################
    # admin-user service account
    ######################################
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    
    ---
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
      namespace: kubernetes-dashboard
    ```

3. 执行安装。
    ```
    kubectl apply -f recommended.yaml
    ```

4. 检查 kubernetes-dashboard 安装情况。
    ```
    kubectl -n kubernetes-dashboard get pods,services
    ```

5. 获取admin-user的token，用于登录kubernetes-dashboard。
    ```
    kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
    ```

6. 复制 token，打开浏览器访问：https://vip:30000，输入token登录。
