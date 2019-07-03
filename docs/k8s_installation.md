# 搭建 k8s 集群

## 准备工作
* 两台 `CentOS 7.4 64位` 服务器，在同一个局域网，一台用作 master 节点，一台用作 node 节点
* k8s `v1.11.2` 编译好的二进制文件
  
  1. 下载k8s文件
      > wget https://github.com/kubernetes/kubernetes/releases/download/v1.11.2/kubernetes.tar.gz
  2. 解压
      > tar -zxvf kubernetes.tar.gz
  3. 下载我们需要的二进制文件
      > ./kubernetes/cluster/get-kube-binaries.sh
  4. 解压下载好的二进制文件
      > cd kubernetes/server/ & tar -zxvf kubernetes-server-linux-amd64.tar.gz
  5. 查看解压后的文件
      > ls kubernetes/server/bin/

      确认包含以下文件，这些文件会在 master 和 node 节点中提供相应的服务：

      * kube-apiserver
      * kube-controller-manager
      * kubectl
      * kubelet
      * kube-proxy
      * kube-scheduler
* 全套的证书文件，用于建立 k8s 内部私密通道，这里使用 `openssl` 生成
  
  1. 新建 `openssl.cnf` 配置文件，内容为：

      ```conf
      [req]
      req_extensions = v3_req
      distinguished_name = req_distinguished_name
      [req_distinguished_name]
      [ v3_req ]
      basicConstraints = CA:FALSE
      keyUsage = nonRepudiation, digitalSignature, keyEncipherment
      subjectAltName = @alt_names
      [alt_names]
      DNS.1 = kubernetes
      DNS.2 = kubernetes.default
      DNS.3 = kubernetes.default.svc
      DNS.4 = kubernetes.default.svc.cluster.local
      DNS.5 = localhost
      DNS.6 = centos-master
      IP.1 = 127.0.0.1
      IP.2 = 10.0.0.1
      IP.3 = 172.17.157.121
      ```
  2. 生成 `CA` 私钥
      > openssl genrsa -out ca-key.pem 2048
  3. 生成 `CA` 证书
      > openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"
  4. 生成 apiserver 私钥 
      > openssl genrsa -out apiserver-key.pem 2048
  5. 生成 apiserver `csr` 文件
      > openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-master" -config openssl.cnf
  6. 生成 apiserver 证书
      > openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 9000 -extensions v3_req -extfile openssl.cnf
  7. 生成 admin 私钥
      > openssl genrsa -out admin-key.pem 2048
  8. 生成 admin `csr` 文件
      > openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=kube-admin"
  9. 生成 admin 证书，用于 `kubectl`、各个 `node` 节点与 apiserver 通信
      > openssl x509 -req -in admin.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out admin.pem -days 9000

## 搭建 master 节点
### 概述

master 节点需要提供以下服务：
  
  * etcd
  * kube-apiserver
  * kube-controller-manager
  * kube-scheduler

### 第一步：安装 etcd `3.2.25`
1. 下载 & 解压
    > wget https://github.com/etcd-io/etcd/releases/download/v3.2.25/etcd-v3.2.25-linux-amd64.tar.gz

    > tar -zxvf etcd-v3.2.25-linux-amd64.tar.gz

2. 拷贝命令
    > cp etcd-v3.2.25-linux-amd64/etcd /usr/bin/  
    > cp etcd-v3.2.25-linux-amd64/etcdctl /usr/bin/

3. 配置 etcd 服务  
 
    1. 创建配置文件，并在配置文件中添加以下内容
        > mkdir /etc/etcd & vim /etc/etcd/etcd.conf

        ```conf
        ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
        ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
        ```
    2. 创建etcd.service文件，并设置为开机启动
        > vim /usr/lib/systemd/system/etcd.service

        ```conf
        [Unit]
        Descriptio=Etcd Server
        After=network.target

        [Service]
        Type=notify
        WorkingDirectory=/etcd/data
        EnvironmentFile=/etc/etcd/etcd.conf
        ExecStart=/usr/bin/etcd

        [Install]
        WantedBy=multi-user.target
        ```

        > systemctl daemon  
        > systemctl enable etcd  
        > systemctl start etcd
    3. 查看服务是否正常启动
        > systemctl status etcd

        显示 `running` 为正常启动

### 第二步：将 `master` 节点所需要的证书文件放到指定位置
1. 将预先准备好的证书文件 `ca.pem`、`apiserver-key.pem`、`apiserver.pem`、`admin.pem`、`admin-key` 放入 `/srv/kubernetes/` 目录下

### 第三步：安装 kube-apiserver
1. 将准备好的二进制文件复制到指定位置，注意替换 kube-apiserver 为正确的路径
    > cp kube-apiserver /usr/bin/

2. 创建配置文件，并在配置文件中添加以下内容
    > mkdir /etc/kubernetes & vim /etc/kubernetes/config

    ```conf
    KUBE_ETCD_SERVERS="--etcd-servers=http://0.0.0.0:2379"

    KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

    KUBE_API_PORT="--insecure-port=8080"

    KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.0.0.0/16"

    KUBE_SERVICE_PORTS="--service-node-port-range=1-65535"

    KUBE_LOGTOSTDERR="--logtostderr=false"

    KUBE_LOG_DIR="--log-dir=/var/log/kubernetes"

    KUBE_LOG_LEVEL="--v=2"

    KUBE_MASTERS="--master=http://0.0.0.0:8080"

    KUBE_ALLOW_PRIV="--allow-privileged=true"
    ```

    > vim /etc/kubernetes/apiserver

    ```conf
    KUBE_API_ARGS="--authorization-mode=RBAC --cloud-provider=external --admission-control=NamespaceLifecycle,LimitRanger,ResourceQuota,ServiceAccount --tls-cert-file=/srv/kubernetes/apiserver.pem --tls-private-key-file=/srv/kubernetes/apiserver-key.pem --client-ca-file=/srv/kubernetes/ca.pem --service-account-key-file=/srv/kubernetes/apiserver-key.pem"
    ```

    `--cloud-provider=external`这个属性是与云平台进行集成时所用，不集成云平台则可以不设置
3. 创建 kube-apiserver.service 文件，并设置为开机启动
    > vim /usr/lib/systemd/system/kube-apiserver.service

    ```conf
    [Unit]
    Description=Kube API Server
    After=etcd.service
    Wants=etcd.service

    [Service]
    Type=notify
    EnvironmentFile=/etc/kubernetes/config
    EnvironmentFile=/etc/kubernetes/apiserver
    ExecStart=/usr/bin/kube-apiserver \
            $KUBE_API_ARGS \
            $KUBE_ETCD_SERVERS \
            $KUBE_API_ADDRESS \
            $KUBE_API_PORT \
            $KUBE_SERVICE_ADDRESSES \
            $KUBE_SERVICE_PORTS \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_DIR \
            $KUBE_LOG_LEVEL
    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```

    > systemctl daemon  
    > systemctl enable kube-apiserver  
    > systemctl start kube-apiserver
4. 查看服务是否正常启动
    > systemctl status kube-apiserver

    显示 `running` 为正常启动

### 第四步：安装 kube-controller-manager
1. 将准备好的二进制文件复制到指定位置，注意替换 kube-controller-manager 为正确的路径
    > cp kube-controller-manager /usr/bin/

2. 创建 `kubeconfig` 配置文件，并添加以下内容，文件中包含了如何放问 apiserver 的信息
    > vim /etc/kubernetes/kube-admin-context.yaml

    ```yaml
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /srv/kubernetes/ca.pem
        server: https://127.0.0.1:6443
      name: default-cluster
    contexts:
    - context:
        cluster: default-cluster
        user: default-admin
      name: default-system
    current-context: default-system
    kind: Config
    preferences: {}
    users:
    - name: default-admin
      user:
        client-certificate: /srv/kubernetes/admin.pem
        client-key: /srv/kubernetes/admin-key.pem
    ```

3. 创建配置文件，并在配置文件中添加以下内容
    > vim /etc/kubernetes/controller-manager

    ```conf
    KUBE_CONTROLLER_MANAGER_ARGS="--kubeconfig=/etc/kubernetes/kube-admin-context.yaml --cloud-provider=external --service-account-private-key-file=/srv/kubernetes/apiserver-key.pem --root-ca-file=/srv/kubernetes/ca.pem"
    ```

4. 创建 kube-controller-manager.service 文件，并设置为开机启动
    > vim /usr/lib/systemd/system/kube-controller-manager.service

    ```conf
    [Unit]
    Description=Kube Controller Manager
    After=etcd.service
    After=kube-apiserver.service
    Requires=etcd.service
    Requires=kube-apiserver.service

    [Service]
    EnvironmentFile=/etc/kubernetes/config
    EnvironmentFile=/etc/kubernetes/controller-manager
    ExecStart=/usr/bin/kube-controller-manager \
            $KUBE_CONTROLLER_MANAGER_ARGS \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_DIR \
            $KUBE_LOG_LEVEL
    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```

    > systemctl daemon  
    > systemctl enable kube-controller-manager  
    > systemctl start kube-controller-manager
5. 查看服务是否正常启动
    > systemctl status kube-controller-manager

    显示 `running` 为正常启动

### 第五步：安装 kube-scheduler
1. 将准备好的二进制文件复制到指定位置，注意替换 kube-scheduler 为正确的路径
    > cp kube-scheduler /usr/bin/

2. 创建配置文件，并在配置文件中添加以下内容
    > vim /etc/kubernetes/scheduler

    ```conf
    KUBE_SCHEDULER_ARGS="--kubeconfig=/etc/kubernetes/kube-admin-context.yaml"
    ```

3. 创建 kube-scheduler.service 文件，并设置为开机启动
    > vim /usr/lib/systemd/system/kube-scheduler.service

    ```conf
    [Unit]
    Description=Kube Scheduler
    After=etcd.service
    After=kube-apiserver.service
    Requires=etcd.service
    Requires=kube-apiserver.service

    [Service]
    EnvironmentFile=/etc/kubernetes/config
    EnvironmentFile=/etc/kubernetes/scheduler
    ExecStart=/usr/bin/kube-scheduler \
            $KUBE_SCHEDULER_ARGS \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_DIR \
            $KUBE_LOG_LEVEL
    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```

    > systemctl daemon  
    > systemctl enable kube-scheduler  
    > systemctl start kube-scheduler
4. 查看服务是否正常启动
    > systemctl status kube-scheduler

    显示 `running` 为正常启动

### 第六步：配置 kubectl 命令行客户端
1. 将准备好的二进制文件复制到指定位置
    > cp kubectl /usr/bin/

2. 为 kubectl 使用的用户设置管理员权限

    创建 k8s `ClusterRoleBind` 配置文件
    > vim kubectl-role-binding.yaml

    ```yaml
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: default-admin-cluster-role-binding
    subjects:
    - kind: User
      name: kube-admin
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: cluster-admin
      apiGroup: rbac.authorization.k8s.io
    ```

    根据配置文件创建 `ClusterRoleBinding`
    > kubectl -f kubectl-role-binding.yaml

3. 配置 kubectl 使用私密通道进行通信
    设置集群访问入口
    > kubectl config set-cluster default-cluster --server=https://127.0.0.1:6443 --certificate-authority=/srv/kubernetes/ca.pem

    设置访问凭证
    > kubectl config set-credentials default-admin --certificate-authority=/srv/kubernetes/ca.pem --client-key=/srv/kubernetes/admin-key.pem --client-certificate=/srv/kubernetes/admin.pem

    创建一个用户
    > kubectl config set-context default-system --cluster=default-cluster --user=default-admin

    指明使用新创建的用户
    > kubectl config use-context default-system

### 第七步：查看 master 节点是否安装完成
执行

> kubectl get componentstatus

结果显示 `controller-manager`、`scheduler`、`etcd-0` 的 `status` 为 `Healthy`，证明 master 节点安装完成

## 搭建 node 节点

每一个 node 节点都需要安装以下服务：

  * docker
  * kubelet
  * kube-proxy

### 第一步：安装 docker `17.06.2-ce`
1. 安装 `yum-utils` 以及其所依赖的软件
    ```shell
    sudo yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2
    ```

2. 添加 docker.repo 到 yum 源列表中
    ```shell
    sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    ```

3. 安装对应版本的 docker
    > yum install docker-ce-17.06.2.ce-1.el7.centos

4. 更改 docker `cgroupdriver` 为 `cgroupfs`
    > vim /etc/docker/daemon.json

    ```json
    {
      "exec-opts": ["native.cgroupdriver=cgroupfs"]
    }
    ```

5. 启动 docker 服务
    > systemctl start docker

6. 查看 docker 是否安装完成
    > docker info

    鉴定以下两个内容是否相同，相同则代表安装完成

      * Server Version: `17.06.2-ce`
      * Cgroup Driver: `cgroupfs`

### 第二步：将 node 节点所需的证书放入指定位置
1. 将预先准备好的证书文件 `ca.pem`、`admin.pem`、`admin-key` 放入 `/srv/kubernetes/` 目录下

### 第三步：安装 kubelet
1. 将准备好的二进制文件复制到指定位置，注意替换 kubelet 为正确的路径
    > cp kubelet /usr/bin/

2. 创建 `kubeconfig` 配置文件，并添加以下内容，文件中包含了如何放问 apiserver 的信息，注意要替换 master 节点的 `ip`
    > vim /etc/kubernetes/kube-admin-context.yaml

    ```yaml
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /srv/kubernetes/ca.pem
        server: https://172.17.157.121:6443
      name: default-cluster
    contexts:
    - context:
        cluster: default-cluster
        user: default-admin
      name: default-system
    current-context: default-system
    kind: Config
    preferences: {}
    users:
    - name: default-admin
      user:
        client-certificate: /srv/kubernetes/admin.pem
        client-key: /srv/kubernetes/admin-key.pem
    ```

3. 创建配置文件，并在配置文件中添加以下内容，注意要替换 master 节点的 `ip`
    > mkdir /etc/kubernetes & vim /etc/kubernetes/config

    ```conf
    KUBE_LOGTOSTDERR="--logtostderr=false"

    KUBE_LOG_DIR="--log-dir=/var/log/kubernetes"

    KUBE_LOG_LEVEL="--v=2"

    KUBE_MASTERS="--master=http://172.17.157.121:8080"

    KUBE_ALLOW_PRIV="--allow-privileged=true"
    ```

    > vim /etc/kubernetes/kubelet

    ```conf
    KUBELET_ARGS="--kubeconfig=/etc/kubernetes/kube-admin-context.yaml --cloud-provider=external --provider-id={{regionId.实例Id}} --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice --cgroup-driver=cgroupfs --enable-server=true --cluster-dns=10.0.0.2 --cluster-domain=cluster.local --pod-infra-container-image=registry.cn-huhehaote.aliyuncs.com/farmer/pause-amd64:3.1 --address=0.0.0.0 --port=10250"
    ```
    
    `--cloud-provider=external`、`--provider-id={regionId.实例Id}}`这两个属性是与云平台进行集成时所用，不集成云平台则可以不设置
4. 创建 kubelet.service 文件，并设置为开机启动
    > vim /usr/lib/systemd/system/kubelet.service

    ```conf
    [Unit]
    Description=Kube Kubelet Server
    After=docker.service
    Requires=docker.service

    [Service]
    EnvironmentFile=/etc/kubernetes/config
    EnvironmentFile=/etc/kubernetes/kubelet
    ExecStart=/usr/bin/kubelet \
              $KUBE_LOGTOSTDERR \
              $KUBE_LOG_DIR \
              $KUBE_LOG_LEVEL \
              $KUBELET_ARGS
    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```

    > systemctl daemon  
    > systemctl enable kubelet  
    > systemctl start kubelet
5. 查看服务是否正常启动
    > systemctl status kubelet

    显示 `running` 为正常启动

### 第二步：安装 kube-proxy
1. 将准备好的二进制文件复制到指定位置，注意替换 kube-proxy 为正确的路径
    > cp kube-proxy /usr/bin/

2. 创建配置文件，并在配置文件中添加以下内容
    > vim /etc/kubernetes/kube-proxy

    ```conf
    KUBE_PROXY_ARGS="--kubeconfig=/etc/kubernetes/kube-admin-context.yaml"
    ```

3. 创建 kube-proxy.service 文件，并设置为开机启动
    > vim /usr/lib/systemd/system/kube-proxy.service

    ```conf
    [Unit]
    Descriptio=Kube Kube-Proxy Server
    After=network.target
    Requires=network.target

    [Service]
    EnvironmentFile=/etc/kubernetes/config
    EnvironmentFile=/etc/kubernetes/proxy
    ExecStart=/usr/bin/kube-proxy \
              $KUBE_LOGTOSTDERR \
              $KUBE_LOG_DIR \
              $KUBE_LOG_LEVEL \
              $KUBE_PROXY_ARGS
    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```

    > systemctl daemon  
    > systemctl enable kube-proxy  
    > systemctl start kube-proxy
4. 查看服务是否正常启动
    > systemctl status kube-proxy

      显示 `running` 为正常启动

## 确认集群安装是否成功
进入到 master 节点服务器中，执行以下命令查看 node 节点信息

> kubectl get node

如果能够查询到有一个 node 节点存在，并且 `status` 为 Ready，则 k8s 基础环境的安装已经完成，接下来可以进行网络设施的搭建

## 安装网络设施
### 概述
k8s 基础环境搭建完成就可以运行一些容器了，可以通过 `Nodeport` 的方式将容器提供的服务暴露出去，使其能够被集群外部访问。通常运行在集群中的容器之间需要进行通信（比如 `微服务` 架构），这一点k8s自己是不支持的，也就是说不同 `node` 节点上面的容器无法互相访问，这个问题可以采用第三方支持的网络解决方案：
* flannel （我们采用）
* calico
除了不同 `node` 节点上的容器无法互相访问的问题，还需要搭建一个 `dns`服务用于解析集群内部的域名，使用域名来找到对应的服务是因为 k8s 集群为容器分配内部 `ip` 是动态随机的（虽然可以固定 `node` 节点的 `ip`，但是 `node` 的数量在可伸缩的情况下也是不定的，所以不推荐这样使用），这个 `dns` 服务主要有两种实现方案：
* coredns （我们采用）
* kube-dns

### 安装 `flannel`
`flannel` 利用 `etcd` 来记录容器的网络配置，这里我们使用与 apiserver 相同的 `etcd`；`flannel` 需要在每一个 `node` 节点上安装，`master` 节点则不需要。

#### 第一步：下载编译好的二进制文件
1. 下载
    > wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
2. 解压
    > tar -zxvf flannel-v0.10.0-linux-amd64.tar.gz
3. 检查文件：`flanneld`、`mk-docker-opts.sh` 这两个文件将会用到

#### 第二步：安装 `flannel`
1. 向 `etcd` 中插入配置数据，注意替换 `--endpoints` 为正确的 `etcd` 访问入口
    etcdctl --endpoints="http://172.17.157.121:2379"  set /coreos.com/network/config '{"Network":"10.0.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'

2. 更改 `docker` 服务启动配置
    > vim /usr/lib/systemd/system/docker.service

    ```conf
    [Unit]
    Description=Docker Application Container Engine
    Documentation=https://docs.docker.com
    After=network-online.target firewalld.service
    Wants=network-online.target

    [Service]
    Type=notify
    # the default is not to use systemd for cgroups because the delegate issues still
    # exists and systemd currently does not support the cgroup feature set required
    # for containers run by docker
    EnvironmentFile=/run/flannel/subnet.env
    ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
    ExecReload=/bin/kill -s HUP $MAINPID
    # Having non-zero Limit*s causes performance problems due to accounting overhead
    # in the kernel. We recommend using cgroups to do container-local accounting.
    LimitNOFILE=infinity
    LimitNPROC=infinity
    LimitCORE=infinity
    # Uncomment TasksMax if your systemd version supports it.
    # Only systemd 226 and above support this version.
    #TasksMax=infinity
    TimeoutStartSec=0
    # set delegate yes so that systemd does not reset the cgroups of docker containers
    Delegate=yes
    # kill only the docker process, not all processes in the cgroup
    KillMode=process
    # restart the docker process if it exits prematurely
    Restart=on-failure
    StartLimitBurst=3
    StartLimitInterval=60s

    [Install]
    WantedBy=multi-user.target
    ```

3. 将准备好的二进制文件和可执行脚本复制到 `node` 节点指定位置，注意替换 `flanneld`、`` 为正确的路径
    > cp flanneld /usr/bin/
    > cp mk-docker-opts.sh /usr/bin/

4. 创建配置文件，并在配置文件中添加以下内容，注意替换 `--etcd-endpoints` 为正确的 `etcd` 访问入口
    > vim /etc/kubernetes/flanneld

    ```conf
    FLANNELD_ARGS="--etcd-endpoints=http://172.17.157.121:2379  --etcd-prefix=/coreos.com/network"
    ```

5. 创建 flanneld.service 文件，并设置为开机启动
    > vim /usr/lib/systemd/system/flanneld.service

    ```conf
    [Unit]
    Description=Flanneld overlay address etcd agent
    After=network.target
    After=network-online.target
    Wants=network-online.target
    Before=docker.service

    [Service]
    Type=notify
    EnvironmentFile=/etc/flannel/flanneld
    ExecStart=/usr/bin/flanneld \
              $FLANNELD_ARGS
    ExecStartPost=/usr/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    RequiredBy=docker.service
    ```

    > systemctl daemon  
    > systemctl enable flanneld  
    > systemctl start flanneld
    > systemctl restart docker
6. 查看服务是否正常启动
    > systemctl status flanneld

    显示 `running` 为正常启动
  
    > ifconfig

    查看 `docker0` 网桥的 `ip` 是否为：10.0.*.1
    查看 `flannel.1` 网桥的 `ip` 是否为：10.0.80.0

### 安装 `coredns`
1. 创建 `coredns.yaml` 文件
    ```yaml
    # Warning: This is a file generated from the base underscore template file: coredns.yaml.base

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: coredns
      namespace: kube-system
      labels:
          kubernetes.io/cluster-service: "true"
          addonmanager.kubernetes.io/mode: Reconcile
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
        addonmanager.kubernetes.io/mode: Reconcile
      name: system:coredns
    rules:
    - apiGroups:
      - ""
      resources:
      - endpoints
      - services
      - pods
      - namespaces
      verbs:
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
        addonmanager.kubernetes.io/mode: EnsureExists
      name: system:coredns
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:coredns
    subjects:
    - kind: ServiceAccount
      name: coredns
      namespace: kube-system
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: coredns
      namespace: kube-system
      labels:
          addonmanager.kubernetes.io/mode: EnsureExists
    data:
      Corefile: |
        .:53 {
            errors
            health
            kubernetes cluster.local in-addr.arpa ip6.arpa {
                pods insecure
                upstream
                fallthrough in-addr.arpa ip6.arpa
            }
            prometheus :9153
            proxy . /etc/resolv.conf
            cache 30
            reload
        }
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: coredns
      namespace: kube-system
      labels:
        k8s-app: kube-dns
        kubernetes.io/cluster-service: "true"
        addonmanager.kubernetes.io/mode: Reconcile
        kubernetes.io/name: "CoreDNS"
    spec:
      # replicas: not specified here:
      # 1. In order to make Addon Manager do not reconcile this replicas parameter.
      # 2. Default is 1.
      # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
      selector:
        matchLabels:
          k8s-app: kube-dns
      template:
        metadata:
          labels:
            k8s-app: kube-dns
          annotations:
            seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
        spec:
          serviceAccountName: coredns
          serviceAccount: coredns
          tolerations:
            - key: node-role.kubernetes.io/master
              effect: NoSchedule
            - key: "CriticalAddonsOnly"
              operator: "Exists"
          containers:
          - name: coredns
            image: coredns/coredns:1.1.3
            imagePullPolicy: IfNotPresent
            resources:
              limits:
                memory: 170Mi
              requests:
                cpu: 100m
                memory: 70Mi
            args: [ "-conf", "/etc/coredns/Corefile" ]
            volumeMounts:
            - name: config-volume
              mountPath: /etc/coredns
              readOnly: true
            ports:
            - containerPort: 53
              name: dns
              protocol: UDP
            - containerPort: 53
              name: dns-tcp
              protocol: TCP
            - containerPort: 9153
              name: metrics
              protocol: TCP
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
                scheme: HTTP
              initialDelaySeconds: 60
              timeoutSeconds: 5
              successThreshold: 1
              failureThreshold: 5
            securityContext:
              allowPrivilegeEscalation: false
              capabilities:
                add:
                - NET_BIND_SERVICE
                drop:
                - all
              readOnlyRootFilesystem: true
          dnsPolicy: Default
          volumes:
            - name: config-volume
              configMap:
                name: coredns
                items:
                - key: Corefile
                  path: Corefile
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: kube-dns
      namespace: kube-system
      annotations:
        prometheus.io/port: "9153"
        prometheus.io/scrape: "true"
      labels:
        k8s-app: kube-dns
        kubernetes.io/cluster-service: "true"
        addonmanager.kubernetes.io/mode: Reconcile
        kubernetes.io/name: "CoreDNS"
    spec:
      selector:
        k8s-app: kube-dns
      clusterIP: 10.0.0.2
      ports:
      - name: dns
        port: 53
        protocol: UDP
      - name: dns-tcp
        port: 53
        protocol: TCP
    ```
2. 使用创建好的 `coredns.yaml` 部署 `coredns` 服务
    > kubectl apply -f coredns.yaml
3. 查看 `coredns` 服务状态
    > kubectl get all -n kube-system

## 使用 `阿里云负载均衡` 暴露公网IP
### 概述
k8s 提供接口与第三放云平台进行整合，整合之后即可创建 type 为 `LoadBalancer` 的 Service，与之整合的云平台则会为你的服务创建一个负载均衡服务，提供公网IP访问。

### 准备工作
* node 节点的 `regionId` 和 `实例Id`
* 阿里云帐号的 `accessKey` 和 `accessSecret`

### 第一步：获取 `accessKey`、`accessSecret` 的base64编码
  > echo -n "accessKey" | base64

  > echo -n "accessKey" | base64

### 第二步：创建 `secret`
1. 创建 `aliyun-secret.yaml` 文件，注意要替换 `accessKey`、`accessSecret` 为对应的 base64 值
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: aliyun-config
      namespace: kube-system
    data: 
      access-key-id: {{accessKey}}
      access-key-secret: {{accessSecret}}
    ```
2. 使用创建好的 `aliyun-secret.yaml` 文件创建 `secret`
  > kubectl apply -f aliyun-secret.yaml

### 第三步：创建 `cloud-controller-manager`
1. 创建 `aliyun-controller-manager.yaml` 文件，注意要替换 `--master` 为正确的非加密的入口（加密还带研究）
    ```yaml
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: cloud-controller-manager
      namespace: kube-system
    spec:
      replicas: 1
      revisionHistoryLimit: 2
      template:
        metadata:
          labels:
            app: cloud-controller-manager
        spec:
          dnsPolicy: Default
          tolerations:
            - key: "node.cloudprovider.kubernetes.io/uninitialized"
              value: "true"
              effect: "NoSchedule"
          containers:
          - image: registry.cn-huhehaote.aliyuncs.com/farmer/alicloud-controller-manager:v0.1.0
            name: cloud-controller-manager
            command:
              - /alicloud-controller-manager
              - --leader-elect=false
              - --allocate-node-cidrs=true
              - --cluster-cidr=10.0.0.0/16
              - --master=172.17.157.121:8080
            env:
              - name: ACCESS_KEY_ID
                valueFrom:
                  secretKeyRef:
                    name: aliyun-config
                    key: access-key-id
              - name: ACCESS_KEY_SECRET
                valueFrom:
                  secretKeyRef:
                    name: aliyun-config
                    key: access-key-secret
    ```
2. 使用创建的 `aliyun-controller-manager.yaml` 文件创建 `cloud-controller-manager`
  > kubectl apply -f aliyun-controller-manager.yaml
3. 查看 `cloud-controller-manager` 是否正常运行
  > kubectl get all -n kube-system

### 第四步：更改 kube-apiserver 服务配置
  > vim /etc/kubernetes/apiserver

  增加参数 `--cloud-provider=external`，并重启 `kube-apiserver` 服务

### 第五步：更改 kubelet 服务配置
  > vim /etc/kubernetes/kubelet

  增加参数 `--cloud-provider=external`、`--provider-id={regionId.实例Id}}`，并重启 `kubelet`
