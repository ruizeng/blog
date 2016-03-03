# 简介

docker是这两年非常火爆的devops工具。本文记录如何搭建安全的私有docker仓库。关于docker技术的介绍，可以参考其[官网](https://www.docker.com/)。

docker官方提供了[docker hub](https://registry.hub.docker.com/)来管理公共镜像源。docker和github一样，托管在其上的镜像都是开放的。有的时候针对私有项目的场景，我们希望在自己的私有服务器上搭建一套docker仓库。本文的搭建环境为ubuntu14.04。

## 第1步：安装依赖

docker registry本身只是python编写的一个web服务，所以，需要先安装python运行环境：

``` bash
sudo apt-get update
sudo apt-get -y install build-essential python-dev libevent-dev python-pip liblzma-dev swig libssl-dev
```
## 第2步：安装及配置docker registry

首先，采用pip工具安装docker registry：

``` bash
sudo pip install docker-registry
```
docker-registry运行所需的配置文件示例在/usr/local/lib/python2.7/dist-packages/config目录下，此处直接复制默认配置：

``` bash
cd /usr/local/lib/python2.7/dist-packages/config
sudo cp config_sample.yml config.yml
```
## 第3步: 启动docker registry服务

docker registry通过gunicorn运行，通过命令行可以直接启动：

``` bash
gunicorn --access-logfile - --debug -k gevent -b 0.0.0.0:5000 -w 1 docker_registry.wsgi:application
```
一般情况下，我们需要将web服务配置为后台模式运行，这里采用upstart工具设置docker registry自启动并后台运行。
首先新建一个目录存放docker registry的log：

``` bash
sudo mkdir /var/log/docker-registry
```
在/etc/init/下新建docker-registry.conf文件，配置其自动启动docker registry：

``` bash
description &quot;Docker Registry&quot;

start on runlevel [2345]
stop on runlevel [016]

respawn
respawn limit 10 5

script
exec gunicorn --access-logfile /var/log/docker-registry/access.log --error-logfile /var/log/docker-registry/server.log -k gevent --max-requests 100 --graceful-timeout 3600 -t 3600 -b localhost:5000 -w 8 docker_registry.wsgi:application
end script
```
配置完成后，就可以通过service命令启动docker registry了：

``` bash
sudo service docker-registry start
```
如果docker仓库是运行在本地环境，那么至此已经成功搭建可使用的docker registry了。
但是如果我们的私有docker仓库希望通过外网也可以访问，那么必须增加一定的安全机制了。

## 第4步：使用nginx代理web请求

nginx作为常用的代理服务器，其性能和功能都很强大。采用nginx实现的web安全机制，可以增加外网访问的安全性。首先安装nginx：

``` bash
sudo apt-get -y install nginx apache2-utils
```
创建用户名及密码：

``` bash
sudo htpasswd -c /etc/nginx/docker-registry.htpasswd USERNAME
```
现在我们在/etc/nginx/docker-registry.htpasswd下新建了一个名字为USERNAME的用户。后续需要增加用户时可以随时执行该命令添加。通过编辑该文件，删除对应的行也可以删除用户。

下面配置nginx让其使用该认证文件，并且代理对docker registry的访问。
新建/etc/nginx/sites-available/docker-registry文件并编辑：

``` 
# For versions of Nginx &gt; 1.3.9 that include chunked transfer encoding support
# Replace with appropriate values where necessary

upstream docker-registry {
 server localhost:5000;
}

server {
 listen 8080;
 server_name your.docker.registry.com;

 # ssl on;
 # ssl_certificate /etc/ssl/certs/docker-registry;
 # ssl_certificate_key /etc/ssl/private/docker-registry;

 proxy_set_header Host       $http_host;   # required for Docker client sake
 proxy_set_header X-Real-IP  $remote_addr; # pass on real client IP

 client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads

 # required to avoid HTTP 411: see Issue #1486 (https://github.com/dotcloud/docker/issues/1486)
 chunked_transfer_encoding on;

 location / {
     # let Nginx know about our auth file
     auth_basic              &quot;Restricted&quot;;
     auth_basic_user_file    docker-registry.htpasswd;

     proxy_pass http://docker-registry;
 }
 location /_ping {
     auth_basic off;
     proxy_pass http://docker-registry;
 }  
 location /v1/_ping {
     auth_basic off;
     proxy_pass http://docker-registry;
 }

}
```
接下来，配置并重启服务：

``` bash
sudo ln -s /etc/nginx/sites-available/docker-registry /etc/nginx/sites-enabled/docker-registry
sudo service nginx restart
```
此时我们从浏览器中已经可以访问ip:8080，会弹出认证页面。输入刚才设置的USERNAME及其密码，就可以看到"\"docker-registry server\""字样。证明已经配置成功了！
但是当前是基于HTTP协议的，传输协议并未进行加密。为了更安全，我们应该采用https。

## 第5步：配置ssl加密传输(https)

如果你有自己的ssl证书，则可以直接使用。否则需要自己制作一个自认证证书：

``` bash
openssl req -newkey rsa:2048 -x509 -nodes -days 3560 -out your-docker-registry.com.crt -keyout your-docker-registry.com.key
```
将生成的key文件和crt文件分别拷贝到/etc/ssl/private/docker-registry和/etc/ssl/certs/docker-registry目录下。

``` bash
sudo cp your-docker-registry.com.crt /etc/ssl/certs/docker-registry
sudo cp your-docker-registry.com.key /etc/ssl/private/docker-registry
```
编辑nginx配置文件/etc/nginx/sites-available/docker-registry，取消其中ssl相关注释，并将监听端口修改为https默认的443端口：

``` 
server {
      listen 443;
      server_name yourdomain.com;

      ssl on;
      ssl_certificate /etc/ssl/certs/docker-registry;
      ssl_certificate_key /etc/ssl/private/docker-registry;
      ...
```
重启nginx，docker registry就支持ssl了！但是遗憾的是docker目前不支持自认证ssl证书的访问，所以针对每台需要访问registry的服务器，我们需要将自己的证书加入到本机的证书池中以通过证书验证。

## 第6步：从其他机器上访问docker registry

将上一步生成的.crt证书拷贝到其他机器的/usr/share/ca-certificates/extra目录下，然后更新证书

``` bash
sudo dpkg-reconfigure ca-certificates
```
选中新增加的证书，然后确认安装即可。

接下来重启docker进程，让其重新加载证书：

``` bash
sudo service docker restart
```
接下来就能够在其他机器上登录到我们的私有registry上了。

``` bash
docker login https://your.docker.registry.com/
```

当出现“Login Succeeded”字样则证明登录成功。

注意：

由于docker依然在高速迭代的过程中，在安装和使用docker及registry时都尽量保持安装的是最新版本，否则可能出现docker和docker registry不兼容的问题而出错。
登录成功后，就可以使用docker的命令了。docker和私有registry的交互会以tag名以registry域名开头来区分,比如我们从官方registry下载ubuntu镜像并上传到我们的私有registry：

``` bash
docker pull ubuntu
docker tag ubuntu your.docker.registry.com/ubuntu
docker push your.docker.registry.com/ubuntu
```
至此，私有docker仓库就搭建完成了！
 