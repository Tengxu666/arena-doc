---
title: 集群部署
weight: 2
---

## Harbor部署

| IP            | User | Password    |
| ------------- | ---- | ----------- |
| 192.168.10.34 | ntx  | P@ssw0rd |

部署harbor镜像仓库,后续集群部署所需镜像使用Harbor提供;

### 安装docker

```bash
# 安装docker
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update
sudo apt-get install docker-ce=5:19.03.15~3-0~ubuntu-focal docker-ce-cli=5:19.03.15~3-0~ubuntu-focal containerd.io

# 用户ntx加到docker组
sudo groupadd docker
# sudo gpasswd -a ntx docker
sudo gpasswd -a ${USER} docker
sudo service docker restart
```

### 安装docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker-compose -v

# 如果安装后docker-compose不可用，创建符号链接到/use/bin
# sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### 下载离线包

下载harbor离线包并解压

```bash
$ sudo mkdir -p /data/harbor/data
$ cd /data/harbor/

# 下载离线包
$ wget https://github.com/goharbor/harbor/releases/download/v2.4.1/harbor-offline-installer-v2.4.1.tgz

# 解压
$ tar zxf harbor-offline-installer-v2.4.1.tgz
# 从压缩包中加载docker镜像
$ docker image load -i harbor/harbor.v2.4.1.tar.gz
```

### 修改配置文件

修改配置文件harbor.yml

```bash
$ cd /data/harbor/harbor
$ cp harbor.yml.tmpl harbor.yml

$ vim harbor.yml
# 修改hostname为harbor节点ip
hostname: 192.168.10.34

# 修改http端口为5000
http:
  port: 5000
  
# 设置admin用户的密码
harbor_admin_password: Harbor12345

# 配置默认数据目录
data_volume: /data/harbor/data
```

### 启动harbor服务

```bash
# 启动harbor
$ cd /data/harbor/harbor
$ sudo ./install.sh --with-chartmuseum

# 停止harbor
# docker-compose down

# 重启
# sudo ./install.sh --with-chartmuseum
```

通过[http://192.168.10.34:5000](http://192.168.10.34:5000)即可访问harbor镜像仓库:


## NFS部署

持久化存储是安装 AI平台 的必备条件。NFS用于为有状态服务动态提供持久化存储。

### nfs服务端

| IP            | User | Password    |
| ------------- | ---- | ----------- |
| 192.168.10.34 | ntx  | P@ssw0rd |


#### 安装nfs server

```bash
# 安装nfs server
sudo apt-get update
sudo apt install nfs-kernel-server -y
sudo sh -c 'echo /mnt/nvme0n1   "*(rw,sync,insecure,no_subtree_check,no_root_squash)" >>  /etc/exports'
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

#### 文件系统配额

开启文件系统配额管理。
安装配额工具：

```bash
$ sudo apt update
$ sudo apt install quota

# 验证安装
$ quota --version
```

安装配额内核模块：

```bash
# 验证是否安装
$ find /lib/modules/`uname -r` -type f -name '*quota_v*.ko*'
# 如果已安装会输出一下内容
Output
/lib/modules/5.4.0-109-generic/kernel/fs/quota/quota_v1.ko
/lib/modules/5.4.0-109-generic/kernel/fs/quota/quota_v2.ko

# 如果未安装，安装linux-image-extra-virtual
$ sudo apt install linux-image-extra-virtual
# 如果上一步安装后因为内核原因还是不能输出期望的内容。安装对应内核的模块
# eg： sudo apt install 'linux-modules-extra-5.4.0-40-generic'
```

修改/etc/fstab，更新文件系统挂载选项：增加`usrquota,grpquota`。

重新挂载文件系统使配置文件生效：

```bash
$ sudo mount -o remount /mnt/nvme0n1
```

允许使用基于用户与用户组的配额管理:

```bash
$ sudo quotacheck -ugm /mnt/nvme0n1

$ sudo quotaon -v /mnt/nvme0n1
```

拷贝安装包并解压：

```bash
# AIPLATEFORM_OFFINE_PATH=/pathtoaipalteform
AIPLATEFORM_OFFINE_PATH=/home/ntx

# 需要将安装包deeparena-installer.tar.gz拷贝到AIPLATEFORM_OFFINE_PATH

# 解压安装包
cd $AIPLATEFORM_OFFINE_PATH
tar -zxvf deeparena-installer.tar.gz

```

部署文件系统配额管理服务：

```bash
# 配额管理服务
cp $AIPLATEFORM_OFFINE_PATH/deeparena-installer/bin/nfs-quota /usr/local/bin/nfs-quota
chmod +x /usr/local/bin/nfs-quota
```

配置服务自启：

```bash
$ sudo tee /etc/systemd/system/fs-quota.service <<-'EOF'
[Unit]
After=network.service

[Service]
ExecStart=/usr/local/bin/nfs-quota -d /mnt/nvme0n1 -p 9088

[Install]
WantedBy=default.target

EOF


$ sudo chmod 664 /etc/systemd/system/fs-quota.service

$ sudo systemctl daemon-reload
# 开机自启
$ sudo systemctl enable fs-quota
# 启动服务
$ sudo systemctl start fs-quota
# 查看服务状态
$ sudo systemctl status fs-quota
```

### nfs客户端

| IP            | User | Password    |
| ------------- | ---- | ----------- |
| 192.168.10.35 | ntx  | P@ssw0rd |


安装nfs client并挂载nfs server：

```bash
# 安装nfs client
# nfs server ip
NFS_SERVER_IP=192.168.10.34
sudo apt install nfs-common
sudo showmount -e $NFS_SERVER_IP
#在nfs客户端创建目录 /mnt/deeparena_nfs
sudo mkdir -p /mnt/deeparena_nfs
#挂载nfs服务端
sudo mount -t nfs $NFS_SERVER_IP:/mnt/nvme0n1 /mnt/deeparena_nfs
```

## Kubernetes安装部署

参考Kubernetes安装管理手册。


## AI平台服务部署

### 基础服务部署

> 提示：
> 如果在非Kubernetes控制节点部署基础服务，请确保服务器已经安装helm和kubectl并且能够访问kubernetes集群api-serve，建议在kubernetes控制节点运行部署脚本。


拷贝安装包并解压：

```bash
# AIPLATEFORM_OFFINE_PATH=/pathtoaipalteform
AIPLATEFORM_OFFINE_PATH=/home/ntx

# 需要将安装包deeparena-installer.tar.gz拷贝到AIPLATEFORM_OFFINE_PATH

# 解压安装包
cd $AIPLATEFORM_OFFINE_PATH
tar -zxvf deeparena-installer.tar.gz

```

部署基础服务：
执行部署脚本过程中需要根据实际情况修改变量，参考一下表格：

| nfs server ip          | 192.168.10.34  |
| ---------------------- | -------------- |
| nfs server path        | /mnt/nvme0n1   |
| elastic password       | P@ssw0rd      |
| redis password         | P@ssw0rd    |
| harbor username        | admin          |
| harbor password        | Harbor12345    |
| nfs server             | 192.168.10.34  |
| k8s control plane node | 192.168.10.102 |

```bash
cd $AIPLATEFORM_OFFINE_PATH/deeparena-installer

# 执行部署脚本
./deploy.sh -d
```

### web服务部署

| IP            | User | Password    |
| ------------- | ---- | ----------- |
| 192.168.10.35 | ntx  | P@ssw0rd |


todo： 小勇


web ui 服务部署

```bash
docker run -it -d -p 3008:3008 --name aiserver-web registry.harbor:5000/resource/ai_server:1.0.0
```

