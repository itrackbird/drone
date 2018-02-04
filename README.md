## 概述

这是关于CI工具[drone](https://github.com/drone/drone)的安装、使用记录，主要实现源码编译、源码编译、下载生成物、远程执行shell命令、微信通知、发布服务等。

### 安装

以下drone的安装主要是基于与Gogs集成，数据库采用sqlite，参考[官方介绍](http://docs.drone.io/install-for-gogs/)。通过`docker-compose`来简化安装步骤，确保Linux主机上安装有docker以及docker-compose，对于其的安装参考网上教程，安装完毕后如下

```shell
[root@izwz9hodl26qzpbbkchctpz /]# docker version
Client:
 Version:       17.12.0-ce
 API version:   1.35
 Go version:    go1.9.2
 Git commit:    c97c6d6
 Built: Wed Dec 27 20:10:14 2017
 OS/Arch:       linux/amd64

Server:
 Engine:
  Version:      17.12.0-ce
  API version:  1.35 (minimum version 1.12)
  Go version:   go1.9.2
  Git commit:   c97c6d6
  Built:        Wed Dec 27 20:12:46 2017
  OS/Arch:      linux/amd64
  Experimental: true
  
[root@izwz9hodl26qzpbbkchctpz /]# docker-compose version
docker-compose version 1.16.1, build 6d1ac219
docker-py version: 2.5.1
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
```

之后，编写`docker-compose.yml`文件来设置安装步骤

```shell
[root@izwz9hodl26qzpbbkchctpz drone]# cat docker-compose.yml
version: '2'

services:
  drone-server:
    image: drone/drone:0.8
    ports:
      - 80:8000
      - 9000
    volumes:
      - /var/lib/drone:/var/lib/drone/
    restart: always
    environment:
      - DRONE_OPEN=true
      - DRONE_HOST=${DRONE_HOST}
      - DRONE_GOGS=true
      - DRONE_GOGS_URL=http://gogs.mycompany.com
      - DRONE_SECRET=${DRONE_SECRET}

  drone-agent:
    image: drone/agent:0.8
    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_SERVER=drone-server:9000
      - DRONE_SECRET=${DRONE_SECRET}
```

`DRONE_HOST`填写主机的IP地址，`DRONE_GOGS_URL`填写Gogs安装的地址，如果端口不是80，则需要添加特定端口号。

> **对于drone的server和agent其他版本安装参考dockerhub镜像版本**