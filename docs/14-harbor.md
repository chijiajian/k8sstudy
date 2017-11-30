# 部署VMWare harbor私有仓库 #

Harbor支持角色访问控制，镜像复制，AD/LDAP集成的私有镜像库。
harbor项目地址：[https://github.com/vmware/harbor/](https://github.com/vmware/harbor/)

根据官方文档进行docker-compose安装

## 下载安装 ##
    curl -x $https_proxy -L https://github.com/docker/compose/releases/download/1.17.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
    
    chmod +x /usr/local/bin/docker-compose

## 验证 ##

    docker-compose --version
    docker-compose version 1.17.0, build ac53b73

## 安装harbor步骤 ##
### 下载和安装harbor ###

    curl -x $https_proxy -OL https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-online-installer-v1.2.2.tgz
    
解压

    tar xvf harbor-online-installer-v1.2.2.tgz 

创建harbor nginx服务器TLS证书

继续使用之前配置过的kubernetes-csr.json签名请求和ca来生成harbor TLS证书

<pre>
<code>
 cfssl gencert \
> -ca=ca.pem \
> -ca-key=ca-key.pem \
> -config=ca-config.json \
> -hostname=127.0.0.1,10.66.0.68,xjharbor.easthope.com \
> -profile=kubernetes \
> kubernetes-csr.json | cfssljson -bare harbor

</code>
</pre> 



> 生成

    harbor-key.pem
    harbor.pem


证书分发

    mkdir -p /etc/harbor/ssl
    mv harbor.pem harbor-key.pem /etc/harbor/ssl


### 配置harbor ###

根据官方说明配置文件harbor.cfg中有两个类别的参数，一个是required parameters，一个是optional parameters

本实验修改的Required parameters参数有

    hostname 不要使用127.0.0.1否则外部客户端无法使用此镜像库
    ui_url_protocol 使用http或https
    db_password 
    ssl_cert
    ssl_cert_key
    存储backend可以使用swift、ceph等。


./install.sh进行安装

...
✔ ----Harbor has been installed and started successfully.----

## 验证： ##
通过域名(或IP地址)登入进行验证 https://xjharbor.domain.com/或https://10.66.0.68  默认的用户名为admin密码为Harbor12345，可以在配置文件中直接修改，也可以网页登入后进行修改。

登入网页后通过-配置管理，可以修改认证模式为LDAP


如果之前使用http协议而没使用https的话，也可以使用./prepare生成相关配置文件，同时docker-compose down然后docker-compose up -d即可


## docker  daemon使用私有镜像库 ##
docker daemon没有使用"-insecure-registry"选项
复制ca证书到/etc/docjer.certs.d/xjharbor.domain.com（或镜像库的IP地址）下

    mkdir -p /etc/docker/certs.d/10.66.0.68
    cp ca.pem /etc/docker/certs.d/10.66.0.68/ca.crt
  
docker登入测试

    docker login 10.66.0.68
    
    Username: admin
    Password: 
    Login Succeeded

修改之前的docker镜像代理，局域网私有镜像库不使用代理

    cat /etc/systemd/system/docker.service.d/http-proxy.conf 
    [Service]
    Environment="HTTP_PROXY=http://10.0.193.33:1080/" "NO_PROXY=localhost,127.0.0.1,10.66.0.68"



推送个镜像进行测试

<pre>
<code>
docker images

REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
gcr.io/google_containers/pause-amd64   3.0                 99e59f495ffa        19 months ago       747kB

docker tag gcr.io/google_containers/pause-amd64:3.0 10.66.0.68/dev/pause-amd64:3.0
docker push 10.66.0.68/dev/pause-amd64:3.0

The push refers to a repository [10.66.0.68/dev/pause-amd64]
5f70bf18a086: Pushed 
41ff149e94f2: Pushed 
3.0: digest: sha256:f04288efc7e65a84be74d4fc63e235ac3c6c603cf832e442e0bd3f240b10a91b size: 939
</code>
</pre>

检查docker已保存的认证信息

    cat ~/.docker/config.json


