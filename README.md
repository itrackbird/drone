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

`DRONE_HOST`填写主机的IP地址，`DRONE_GOGS_URL`填写Gogs安装的地址，如果端口不是80，则需要添加特定端口号，`DRONE_SECRET`填写设置的密码。

> **对于drone的server和agent其他版本安装参考dockerhub镜像版本**

### 使用

主要实现软件开发过程中CI工作流程，通过在项目根目录下添加`.drone.yml`文件，编写规则类似`docker-compose`，以下是一个Go项目编译的简单示例

```shell
[root@izwz9hodl26qzpbbkchctpz wechat]# tree -a .
.
├── Dockerfile
├── .drone.yml
├── main.go
└── README.md

0 directories, 4 files

[root@izwz9hodl26qzpbbkchctpz wechat]# cat main.go
package main

import "github.com/gin-gonic/gin"

func main() {
        r := gin.Default()
        r.GET("/ping", func(c *gin.Context) {
                c.JSON(200, gin.H{
                        "message": "pong",
                })
        })
        r.Run() // listen and serve on 0.0.0.0:8080
}

[root@izwz9hodl26qzpbbkchctpz wechat]# cat Dockerfile
FROM alpine
ENV a 1
ENV b 22
CMD [ "echo", "hello, world!"]
```

其中`.drone.yml`内容如下

```shell
[root@izwz9hodl26qzpbbkchctpz wechat]# cat .drone.yml
workspace:
  base: /go
  path: src/wechat

pipeline:
  build:
    image: golang
    commands:
      - go get
      - go build
      - echo "hello, world!"
      - echo "hello, again!"

  publish:
    image: plugins/docker
    repo: registry.com/pzm/wechat
    tags: [ 1.0, 1.0.1, latest ]
    registry: registry.com
    
  scp:
    image: appleboy/drone-scp
    host: host.example.com
    username: user
    password: passwd
    port: 22
    target: /files
    source: ./wechat
    when:
      status: [ success ]

  ssh:
    image: appleboy/drone-ssh
    host:
      - example.com
    username: user
    password: passwd
    port: 22
    script:
      - echo hello
      - echo world
    when:
      status: [ success ]
  wechat:
    image: clem109/drone-wechat
    corpid: corpid
    corp_secret: corp_secret
    agent_id: agent_id
    title: ${DRONE_REPO_NAME}
    description: "Build Number: ${DRONE_BUILD_NUMBER} 部署失败. ${DRONE_COMMIT_AUTHOR} please fix. Check the results here: ${DRONE_BUILD_LINK} "
    msg_url: ${DRONE_BUILD_LINK}
    btn_txt: btn
    when:
      status: [ success ]
```

其中`workspace`设置镜像工作目录为`/go/src/wechat`，如果不设置为这样在编译过程中报`GOPATH`报错，具体参考Go语言编译使用教程。

![drone](C:\Users\Administrator\Desktop\drone\drone.png)

其中，对于微信通知设置，登录[微信官网](https://work.weixin.qq.com/)注册企业账号，之后学习官方微信的使用教程，在`我的企业`下获取`corpid`，在`企业应用`下注册一个应用获取`AgentId`、`Secret`后，最后在`.drone.yml`的wechat工作流中填写对应值即可。其他场景使用参考官方使用教程，不在此做介绍。

对于drone的docker插件使用，如果是使用docker官方的registry镜像仓库项目，那么需要重新编译`plugins/docker`镜像，添加registry对应主机IP信任，不然会出现如下报错

```shell
Get https://registry.com:5000/v2/: http: server gave HTTP response to HTTPS client
```

```shell
[root@izwz9hodl26qzpbbkchctpz registry]# tree
.
├── daemon.json
└── Dockerfile

0 directories, 2 files
[root@izwz9hodl26qzpbbkchctpz registry]# cat Dockerfile
FROM plugins/docker
COPY daemon.json /etc/docker/daemon.json
[root@izwz9hodl26qzpbbkchctpz registry]# cat daemon.json
{
  "registry-mirrors": ["https://xxxxmirror.aliyuncs.com"],
  "metrics-addr": "127.0.0.1:9323",
  "experimental": true,
  "insecure-registries": ["registry.com:5000"]
}
```

之后，就可以使用该插件来编译镜像以及推送镜像。

### FAQ

1.使用`plugins/docker`推送较大镜像时任务卡住，无法退出，使用环境为1G1核的主机！

先不确定是主机的性能问题还是软件Bug，更换alpine镜像后推送成功，暂做记录！