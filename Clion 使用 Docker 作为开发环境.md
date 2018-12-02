---
title: Clion 如何使用 Docker 作为开发环境
date: 2018-12-02 18:18:00
categories:
- 技术
tags: 
- c++
- clion
- docker
- imhuwq
---

有时候你可能想用 Docker 作为 C++ 项目的开发环境，就像 Python 用 Pyenv 作为开发环境一样。  
本文就介绍了 Clion 的实现方式，实际体验效果非常令人满意，除了 Debug 的时候稍微麻烦一点(要多敲一个命令)。

<!-- more -->

## 一. 基本思路
Clion 可以通过 ssh 使用远程服务器作为开发环境，我们把 Docker Container 也当做一个远程服务器来看待就好了。   
此时，我们只需要解决以下几个问题：  

1.1 如何跑起来一个有运行环境的 Container?  

    - 如何在 Container 里面搭建环境？(Dockerfile)  

1.2 Clion 如何连接到 Container？  

    - Container 的 IP 地址是什么？(docker-compose networks)   
    - Container 如何启动 SSH 服务？(docker-compose command)  
    - Container 如何常驻？(docker-compose command)  

1.3 Clion 如何使用 Container 里面的环境？  

    - 如何使用 Container 里面的 Cmake 和各种依赖库？(Clion 设置)  
    - 如何远程 Debug ？(Container gdb server 和 Clion 远程 Debug 设置)  

1.4 本地和 Container 如何及时进行文件同步  

    - 本地源代码如何及时推送到 Container 里面？(Docker volume)  
    - Container 里面程序的输出如何及时拉取到本地？(Docker volume)  

## 二. 实现细节  
接下来逐个解决上述几个问题。  
### 2.1 如何跑起来一个有运行环境的 Container? 
直接用 Dockerfile 安装好必要的依赖，确认安装好 ssh server 和 gdb server。  
建议创建一个非 root 用户，并且赋予这个用户不用密码进行 sudo 的权限。  

```Dockerfile
FROM ubuntu:14.04
MAINTAINER imhuwq "imhuwq@gmail.com"

RUN apt-get update && apt-get install -y openssh-server gdb gdbserver

# 创建 deploy 用户并且赋予不用密码进行 sudo 的权限
RUN echo "#!/bin/bash\nadduser deploy << EOF\npassword\npassword\ndeploy\n\n\n\nY\nEOF" >> create_deploy.sh && \
    chmod 755 create_deploy.sh && \
    ./create_deploy.sh && \
    gpasswd -a deploy sudo && \
    rm create_deploy.sh && \
    chmod 644 /etc/sudoers && \
    echo "deploy ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# 执行其它命令以创建必要的环境，不再赘述
RUN ...

USER deploy
# 创建项目源码目录，这个目录将成为 Container 里面构建和执行的工作区
RUN mkdir -p /home/deploy/my-project
WORKDIR /home/deploy/my-project
ENV LC_ALL C.UTF-8

```

这份 Dockerfile 除了安装必要环境和创建用户之外，还创建了一个工作区文件夹 `/home/deploy/my-project`。  
之后我们会把源代码同步到这个文件夹里面。  

### 2.2 Clion 如何连接到 Container？  
要想 Clion 连接到 Container，必须要获知 Container 的 IP 地址，并且要启动 Container 里面的 SSH Server。当然，Container 还得常驻。  
这三个要求都能用 docker-compose 实现。  

```YML
version: "3.4"

x-defaults: &default
  restart: unless-stopped
#  使用当前目录的 Dockerfile 来构建 docker 镜像
  build: .
  volumes:
# 把当前目录(源代码目录) mount 到 docker container 的特定目录，那个目录就是 docker 环境里面进行编译的工作区间
    - .:/home/deploy/my-project
  networks:
    - default

services:
  my-project:
    <<: *default
    container_name: my-project-dev
    hostname: "my-project"
    user: deploy
    working_dir: /home/deploy/my-project
#   需要改变 security_opt， 不然 gdb server 会跑不起来
    security_opt:
      - seccomp:unconfined
#   开启 ssh 服务，这样 clion 就能通过 ssh 连接进来了
#   同时通过 tailf 命令保持 container 不要退出的状态
    command:
      bash -c "sudo service ssh restart && tail -f /dev/null"

# 手动配置网络， 这样就有固定的 ip 了
networks:
  default:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.129.2.0/24

```

这个 docker-compose.yml 文件指定了 network 的子网段，并且以 `sudo service ssh restart && tail -f /dev/null` 的启动命令来同时实现了“开启 SSH Server” 和“保持 Container 常驻”的功能。  
通过 Docker Volume 把当前目录 mount 到了 Container 里面的 `/home/deploy/my-project` 文件夹，实现实时同步代码的功能。    
`security_opt` 的选项是必要的，否则会启动不了 gdb server。  

### 2.3 Clion 如何使用 Container 里面的环境并及时文件同步？  
Clion 本身就支持 SSH 到远程服务器作为开发环境，也支持使用 GDB 进行远程调试，所以主要是怎么对 Clion 进行设置。  
首先我们把 Docker Container 跑起来： 

```bash
docker-compose build
docker-compose up -d
```

在 Clion `Settings-Build,Execution,Deployment-Toolchains` 页面先新建一个 Toolchains 设置，名字叫 **my-project** 吧，类型选 **Remote Host**。 Credential 设置里面填入 Container 的 IP， 如果使用上述的 docker-compose.yml 来启动 Container 的话，IP 一般是 `172.129.2.2`。端口、用户名和密码按自己创建 Docker 镜像的时候来设置。由于我是从源码编译的 Cmake，所以需要变更一下 Cmake 地址为 `/usr/local/bin/cmake`。大家自己会意，按情况来改就好。  
![添加 ToolChain](http://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/clion-using-docker-as-dev-env/1.png)

创建 Toolchains 配置后，在 `Settings-Build,Execution,Deployment-CMake` 页面的 **Toolchain** 下拉菜单里面选择刚才创建的 `my-project`。 然后点击 **Apply** 保存到目前为止的配置。
![使用 ToolChain](http://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/clion-using-docker-as-dev-env/2.png)

由于我们使用 Docker 的 Volume 进行文件同步，所以不再需要使用 Clion 的 SFTP 了。这个可以在 `Settings-Build,Execution,Deployment-Deployment` 里面进行更改。  
当 Clion 成功连接到 Container 后，会自动创建一个对应于 ToolChain 名字的 Deployment 配置。我们需要对它进行微调， 把本地代码目录(比如说 `/home/john/git/github/my-project`) Map 到 Container 里面的工作目录 `/home/deploy/my-project`，然后禁用 Clion 使用 SFTP 对 Container 里面的工作目录进行同步。   
![对应 ToolChain 的 Deployment 配置](http://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/clion-using-docker-as-dev-env/3.png)
默认 Connection 配置不用更改。  

![调整 Mappings](http://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/clion-using-docker-as-dev-env/4.png)
把本地 `/home/john/git/github/my-project` 映射到 Container 里面的 `/home/deploy/my-project`。  

![调整 Exclude Paths](http://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/clion-using-docker-as-dev-env/5.png)
在添加 Exclude Path 的时候，类型选 SFTP，并设置为 Container 里面的 `/home/deploy/my-project`。  

这样我们就直接使用 Docker Volume 进行实时的文件同步了，比 SFTP 更方便。  

### 2.4 Clion 如何使用 Container 来远程 Debug？ 
基本思路是在 Container 里面跑起来 gdb server，然后使用 Clion 的 **Remote GDB Debug** 来进行常规的断点等操作。  
首先在 Clion 右上角的 **RUN/Debug Configurations** 添加 GBD 远程 Debug 的配置：  
![Remote GDB Debug](http://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/clion-using-docker-as-dev-env/6.png)

然后在 Container 里面启动 GDB Server：  

```bash
gdbserver :1234 cmake-build-debug/my_project_exec arg1 arg2 arg_etc
```
这里其实就是在 Container 里面运行构建的程序，并且把参数也一并输入。  
执行后，会有类似的输出：   

```bash
Process cmake-build-debug/my_project_exec created; pid = 7252
Listening on port 1234
Remote debugging from host 172.129.2.2
```

这时候，从 Clion 右上角的 **RUN/Debug Configurations** 下拉菜单中选择 **my-project**， 然后点击 Debug 的 Icon 就能像往常一样进行 Debug 了。

## 三. 进一步优化
对于一些很常见的操作，我们可以写出一个 Shell 脚本：  

```bash
#!/usr/bin/env bash
# docker.sh 

action=$1

if [ $action = "enter" ]; then
    docker exec -it my-project-dev /bin/bash
elif [ $action = "build" ]; then
    docker-compose build
elif [ $action = "up" ]; then
    docker-compose up -d
    echo "172.129.2.2:22"
elif [ $action = "down" ]; then
    docker-compose down
elif [ $action = "debug" ]; then
    docker-compose exec my-project sh -c "gdbserver :1234 cmake-build-debug/my_project_exec ${*:2}"
fi

```

如此，我们想要 Debug 的话，只需要执行 `./docker.sh debug arg1 arg2 arg_etc` 就好了， 方便很多。


## 四. 总结
Clion 使用 Docker 作为开发环境，本质上就是把 Container 当做一个远程服务器来配置。  
由于 Docker 可以使用 Volume 来进行文件同步，所以可以禁用 SFTP，体验更好。  
