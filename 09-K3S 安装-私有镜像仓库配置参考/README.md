Video Reference：[09-安装-私有镜像仓库配置参考](https://www.bilibili.com/video/BV16m4y1U7LS/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[09-安装-私有镜像仓库配置参考](https://github.com/kingsd041/k3s-tutorial/tree/main/09-安装-私有镜像仓库配置参考)

# 1 私有镜像仓库配置参考

K3s 默认使用 `containerd` 作为容器运行时，所以在 docker 上配置镜像仓库是不生效的[ 除非将容器运行时环境设置为Docker，本篇以`containerd`作为容器运行时环境 ]

K3s registry 配置目录为： `/etc/rancher/k3s/registries.yaml`。K3s 启动时，K3s 会检查 `/etc/rancher/k3s/` 中是否存在 `registries.yaml` 文件，并指示 `containerd` 使用文件中定义的镜像仓库。如果你想使用一个私有的镜像仓库，那么你需要在每个使用镜像仓库的节点上以 `root` 身份创建这个文件。

请注意，server 节点默认是可以调度的。如果你没有在 `server` 节点上设置污点，那么将在它们上运行工作负载，请确保在每个 `server` 节点上创建 `registries.yaml` 文件。

## 1.1 镜像仓库配置文件

K3s 镜像仓库配置文件由两大部分组成：`mirrors` 和 `configs`

- Mirrors 是一个用于定义专用镜像仓库的名称和 endpoint 的指令
- Configs 部分定义了每个 mirror 的 TLS 和证书配置。对于每个 mirror，你可以定义`auth`和`/`或`tls`

`containerd` 使用了类似 `Kubernetes` 中 `svc` 与 `endpoint` 的概念，`svc` 可以理解为访问名称，这个名称会解析到对应的 `endpoint` 上。 也可以理解 mirror 配置就是一个反向代理，它把客户端的请求代理到 `endpoint` 配置的后端镜像仓库。`mirror` 名称可以随意填写，但是必须符合IP或域名的定义规则。并且可以配置多个 `endpoint`，默认解析到第一个 `endpoint`，如果第一个 `endpoint` 没有返回数据，则自动切换到第二个 `endpoint`，以此类推。

示例：

```yaml
# mirrors 可以定义多个私有镜像仓库的名称
mirrors:
  "172.31.6.200:5000": # 私有镜像仓库的名称
    endpoint:
      - "http://172.31.6.200:5000"
      - "http://x.x.x.x:5000"
      - "http://y.y.y.y:5000"
  "rancher.ksd.top:5000": # 私有镜像仓库的名称
    endpoint:
      - "http://172.31.6.200:5000"
  "docker.io": # 私有镜像仓库的名称
    endpoint:
      - "https://fogjl973.mirror.aliyuncs.com"
      - "https://registry-1.docker.io"
# 类似于Kubernetes 的Service和Endpoint的关系

# configs 配置私有镜像仓库的相应的用户名和密码以及证书信息
configs:
  "172.31.6.200:5000":
    auth:
      username: admin
      password: Harbor@12345
    tls:
      cert_file: /home/ubuntu/harbor2.kingsd.top.cert
      key_file:  /home/ubuntu/harbor2.kingsd.top.key
      ca_file:   /home/ubuntu/ca.crt
```

可以通过 `crictl pull 172.31.6.200:5000/library/alpine` 和 `crictl pull rancher.ksd.top:5000/library/alpine` 获取到镜像，但镜像都是从同一个仓库获取到的。

```shell
root@rancher-server:/etc/rancher/k3s# systemctl restart k3s.service
root@rancher-server:/etc/rancher/k3s# crictl pull 172.31.6.200:5000/library/alpine
Image is up to date for sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
root@rancher-server:/etc/rancher/k3s# crictl pull rancher.ksd.top:5000/library/alpine
Image is up to date for sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
```

## 1.2 搭建 Harbor 私有镜像仓库

部署Harbor私有镜像仓库

```shell
# Step1: 安装Docker容器运行时环境
# 在 K3s 节点上安装 Docker。可以使用 Rancher 的一个Docker 安装脚本来安装 Docker：
$ curl https://releases.rancher.com/install-docker/19.03.sh | sh

# Step2: 配置镜像加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://po13h3y1.mirror.aliyuncs.com","http://hub-mirror.c.163.com","https://mirror.ccs.tencentyun.com","http://f1361db2.m.daocloud.io"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker && sudo systemctl enable docker

# Step3: 安装Docker-Compose软件
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.13.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Step4: 使用Docker-Compose临时搭建一套Harbor
$ wget https://github.com/goharbor/harbor/releases/download/v2.6.2/harbor-offline-installer-v2.6.2.tgz
$ tar -zxvf harbor-offline-installer-v2.6.2.tgz && cd harbor

$ cp -av harbor.yml.tmpl harbor.yml
$ vim harbor.yml
hostname: reg.kubesphere.com
https: # 后续进行编写
harbor_admin_password: Harbor12345
```

#### Harbor 证书的TLS创建

```shell
$ mkdir -pv harbor/ssl ; cd harbor/ssl
$ cat > cfssl.sh <<EOF
#!/bin/bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
mv cfssl_linux-amd64          /usr/bin/cfssl
mv cfssljson_linux-amd64      /usr/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
EOF
$ bash cfssl.sh
$ cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
$ cat > ca-csr.json <<EOF
{
        "CN": "kubernetes",
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
                "C": "CN",
                "L": "Beijing",
                "ST": "Beijing"
            }
        ]
}
EOF
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

$ cat > reg.kubesphere.com-csr.json <<EOF
{
    "CN": "reg.kubesphere.com",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes reg.kubesphere.com-csr.json | cfssljson -bare reg.kubesphere.com
```

2、Harbor启用HTTPS

```properties
$ vim ./harbor/harbor.yml
  https:
    port:443
    certificate: /root/harbor/ssl/reg.kubesphere.com.pem
    private_key: /root/harbor/ssl/reg.kubesphere.com-key.pem
```

#### 部署 Harbor 镜像仓库

3、重新配置并部署Harbor

```bash
$ ./prepare
# ./install.sh 
$ docker-compose down
$ docker-compose up -d
$ docker-compose ps -a 
      Name                     Command                  State                                          Ports                                   
-----------------------------------------------------------------------------------------------------------------------------------------------harbor-core         /harbor/entrypoint.sh            Up (healthy)                                                                              
harbor-db           /docker-entrypoint.sh 96 13      Up (healthy)                                                                              
harbor-jobservice   /harbor/entrypoint.sh            Up (healthy)                                                                              
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp                                                  
harbor-portal       nginx -g daemon off;             Up (healthy)                                                                              
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:80->8080/tcp,:::80->8080/tcp,                                      
                                                                    0.0.0.0:443->8443/tcp,:::443->8443/tcp                                     
redis               redis-server /etc/redis.conf     Up (healthy)                                                                              
registry            /home/harbor/entrypoint.sh       Up (healthy)                                                                              
registryctl         /home/harbor/start.sh            Up (healthy)
```

浏览器访问：`http:s//<IP地址>`也可以修改主机的host文件将`IP地址 域名进行绑定`。可以查看浏览器的证书。

默认用户名：`admin`

默认密码：`Harbor12345`

![img](https://cdn.nlark.com/yuque/0/2022/png/2555283/1669881669365-9321758f-4cef-4040-a8c6-bd4b5e96a141.png)

Harbor 镜像仓库可以正常使用，并且创建一个 `k3s` 的项目

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1680010691601-8e10c41a-3e8a-43ea-b9c7-a7f41c2a29b5.png)

```shell
# 上传镜像
$ cat /etc/docker/daemon.json
{
      "registry-mirrors": ["https://si7y70hh.mirror.aliyuncs.com"],
      "insecure-registries": ["reg.kubesphere.com","10.0.0.53"]
}

$ systemctl daemon-reload ; systemctl restart docker
$ docker login reg.kubesphere.com
Username: admin
Password: 123456

Login Succeeded

# 上传镜像
$ docker pull nginx && docker tag nginx:latest reg.kubesphere.com/k3s/nginx:latest
$ docker push reg.kubesphere.com/k3s/nginx:latest
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1680011458468-8b7a3793-9cf6-4c08-9ed8-f5429023ab58.png)

#### 基础环境

```shell
# K3s Master
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN=rancher sh -s -

$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   38s   v1.21.14+k3s1

# 查看StorageClass存储类
$ kubectl get sc
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  42s
```

## 1.3 使用 TLS

#### 证书颁发机构颁发的证书

```yaml
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "reg.kubesphere.com":
    endpoint:
      - "https://reg.kubesphere.com"
configs:
  "reg.kubesphere.com":
    auth:
      username: admin
      password: 123456
EOF
# 重启K3s服务
$ systemctl restart k3s

# 拉取相应的镜像文件
$ crictl pull reg.kubesphere.com/k3s/nginx:latest
```

#### 自签名证书

```yaml
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "reg.kubesphere.com":
    endpoint:
      - "https://reg.kubesphere.com"
configs:
  "reg.kubesphere.com":
    auth:
      username: admin
      password: 123456
    tls:
      cert_file: /home/harbor/reg.kubesphere.com.cert
      key_file:  /home/harbor/reg.kubesphere.com.key
      ca_file:   /home/harbor/ca.crt
EOF

# 重启K3s服务
systemctl restart k3s
```

## 1.4 不使用 TLS（Http registry）

在没有 TLS 通信的情况下，需要为 endpoints 指定`http://`，否则将默认为 `https`

```yaml
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "172.31.19.227:5000":
    endpoint:
      - "http://172.31.19.227:5000"
EOF

# 重启K3s服务
systemctl restart k3s
```

`K3s` 将会在 `/var/lib/rancher/k3s/agent/etc/containerd/config.toml` 中为 `containerd` 生成  `config.toml`。如果要对这个文件进行高级设置，你可以在同一目录中创建另一个名为  `config.toml.tmpl`  的文件，此文件将会代替默认设置。

## 1.5 配置 Mirror（镜像加速器）

```yaml
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "docker.io":
    endpoint:
      - "https://po13h3y1.mirror.aliyuncs.com"
      - "http://hub-mirror.c.163.com"
      - "https://mirror.ccs.tencentyun.com"
      - "https://registry-1.docker.io"
      - "http://f1361db2.m.daocloud.io"
EOF

# 重启K3s服务
systemctl restart k3s
```

## 1.6 完整模板示例（Containerd）

```yaml
mirrors:
  "harbor.kingsd.top":
    endpoint:
      - "https://harbor.kingsd.top"
  "harbor2.kingsd.top":
    endpoint:
      - "https://harbor2.kingsd.top"
  "172.31.19.227:5000":
    endpoint:
      - "http://172.31.19.227:5000"
  "docker.io":
    endpoint:
      - "https://fogjl973.mirror.aliyuncs.com"
      - "https://registry-1.docker.io"

configs:
  "harbor.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345

  "harbor2.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345
    tls:
      cert_file: /home/ubuntu/harbor2.kingsd.top.cert
      key_file:  /home/ubuntu/harbor2.kingsd.top.key
      ca_file:   /home/ubuntu/ca.crt
```

## 1.7 配置 Containerd

K3s 将会在`/var/lib/rancher/k3s/agent/etc/containerd/config.toml`中为 `containerd` 生成 `config.toml`。[ 建议不要直接对 containerd 的配置文件进行修改，格式十分的严格 ]

如果要对这个文件进行高级设置，你可以在同一目录中创建另一个名为 `config.toml.tmpl` 的文件，此文件将会代替默认设置。

`config.toml.tmpl`将被视为 Go 模板文件，并且`config.Node`结构被传递给模板。[此模板](https://github.com/k3s-io/k3s/blob/master/pkg/agent/templates/templates.go#L16-L32)示例介绍了如何使用结构来自定义配置文件。