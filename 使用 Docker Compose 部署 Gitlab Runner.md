---
title: 使用 Docker Compose 部署 Gitlab Runner
date: 2023-07-29 17:55:00
categories:
- 技术
tags:
- docker
- gitlab
- devops
---
本文目标：
- 一台主机部署多个 Gitlab Runner
- 实现互相隔离且互不影响的 Gitlab Runner 环境
- 使用 `compose.yaml` 持久化配置和数据，快速复制部署
- 在 Gitlab Runner 中使用 `docker`, `helm` 以及 `kubectl` 

前置要求：
- 宿主机上安装了 `docker` 和 `docker compose`
- 【可选】宿主机上安装了 `kubectl` 和 `helm`

方案限制：
- 只能使用 shell 作为 Gitlab Executor
	- 由于 Gitlab Runner 本身运行在 docker container 中，所以几乎没有影响
<!--more-->

### 一. 构建 Gitlab Runner 运行环境镜像
首先使用以下 `Dockerfile` 来构建一个镜像，作为 Gitlab Runner 的运行环境。  
这个 `Dockerfile` 主要做的是这么几件事情：
- 安装一些常用的系统工具，比如 `git`, `vim` , `ping`, `wget`, `curl` 等
- 安装 `docker`、 `docker compose`、 `kubectl`、 `helm` 和 `gitlab runner`
- 添加 `gitlan runner` 用户并切换到该用户
	- Gitlab Runner 服务一般以 `gitlab runner` 用户启动

```Dockerfile
# This is the image for gitlab runner based on ubuntu:22.04, which adds the following features:
# 1. Install git, git lfs, vim, curl, ping etc;
# 2. Install docker ce, docker compose
# 3. Install kubectl;
# 3. Install helm;
# 4. Install gitlab runner.

FROM ubuntu:22.04

# install git, etc.
RUN apt update && \
    apt upgrade -y && \
    apt install -y sudo vim git git-lfs net-tools iputils-ping dnsutils mtr wget curl tree && \
    rm -rf /var/lib/apt/lists/*

# install docker ce and compose
RUN sudo apt update && \
    sudo apt install -y ca-certificates curl gnupg && \
    sudo install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    sudo chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    sudo rm -rf /var/lib/apt/lists/*

RUN sudo apt update && \
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin && \
    sudo rm -rf /var/lib/apt/lists/*

# install kubectl
RUN sudo apt update && \
    sudo apt install -y ca-certificates curl && \
    curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg && \
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
    sudo apt update && \
    sudo apt install -y kubectl && \
    sudo rm -rf /var/lib/apt/lists/*

# install helm
RUN curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null && \
    sudo apt install apt-transport-https --yes && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list && \
    sudo apt update && \
    sudo apt install -y helm && \
    sudo rm -rf /var/lib/apt/lists/*

# install gitlab runner
RUN curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash && \
    sudo apt install gitlab-runner && \
    sudo rm -rf /var/lib/apt/lists/*

USER root
RUN echo 'gitlab-runner ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
    usermod -aG docker gitlab-runner

USER gitlab-runner
WORKDIR /home/gitlab-runner
```

执行以下命令来构建镜像:
```shell
docker build . -f Dockerfile -t gitlab-runner:v0.1.0
```

### 二. 设置 Gitlab Runner 启动参数

然后使用以下 `docker-compose.yaml` 来启动 Gitlab Runner。这个 `docker-compose.yaml` 主要做了这么几件事情：
- 复制宿主机的 `helm`、`kubectl`、`docker` 配置到容器中并持久化
- 持久化容器中的 `gitlab runner` 配置和数据
- 把宿主机上的 `docker` 透传给容器里面
	- 容器里面的 `docker` (容器/镜像)和宿主机互通
- 启动两个 service：
	- `init`, 用来做文件初始化操作
	- `runner`，用来运行 `gitlab runner`
		- 通过 `tail -f /dev/null` 常驻容器
		- 注意设置 `init: true` 否则 `tail` 进程不会回收僵尸进程
	- `init` 通过 `/shared/.init_done` 通知 `runner` 准备工作是否完成

```yaml
# docker-compose.yaml
version: '3.9'
x-base-config: &base-config
  image: gitlab-runner:v0.1.0
  volumes:
  - shared:/shared
  - helm-config:/home/gitlab-runner/.config/helm/
  - docker-config:/home/gitlab-runner/.docker/
  - kubectl-config:/home/gitlab-runner/.kube/
  - gitlab-runner-config:/etc/gitlab-runner/
  - ${DOCKER_SOCKS}:/var/run/docker.sock
  - ${KUBECTL_CONFIG}:/tmp/.kube_config
  - ${HELM_CONFIG}:/tmp/.helm_config
  - ${DOCKER_CONFIG}:/tmp/.docker_config
services:
  init:
    <<: *base-config
    image: harbor.cvrgo.com/dds/gitlab-runner:v0.1.0
    user: root
    restart: on-failure
    command: 
      - /bin/bash
      - -c
      - |
        rm -rf /shared/.init_done
        cd /home/gitlab-runner
        mkdir -p .kube && cp -r /tmp/.kube_config/* ./.kube/
        mkdir -p .docker && cp -r /tmp/.docker_config/* ./.docker/
        mkdir -p .config/helm && cp -r /tmp/.helm_config/* ./.config/helm/
        chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.kube
        chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.config
        chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.docker
        helm plugin install https://github.com/chartmuseum/helm-push
        ls -al
        touch /shared/.init_done
  runner:
    <<: *base-config
    image: harbor.cvrgo.com/dds/gitlab-runner:v0.1.0
    restart: on-failure
    init: true
    command:
      - /bin/bash
      - -c
      - | 
        sleep 1
        echo "Wait for initialization"
        while [[ ! -e /shared/.init_done ]]; do
          echo "Sleeping for 1 second..."
          sleep 1
        done

        sleep 3
        echo "setup config"
        sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.kube
        sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.docker
        sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.config

        echo "check docker"
        docker --version
        docker compose version
        echo ""

        echo "check environments"
        echo "check kubectl"
        kubectl get nodes
        echo ""

        echo "check helm"
        helm repo update
        helm repo list
        echo ""

        echo "start gitlab-runner"
        sudo gitlab-runner -l info start
        tail -f /dev/null

volumes:
  shared:
  helm-config:
  docker-config:
  kubectl-config:
  gitlab-runner-config:
```

### 三. 启动镜像
有了以上准备后，就可以启动容器并进入其中配置 Gitlab Runner 了。  
为了方便在同一台宿主机上启动多个 Gitlab Runner，我们给每个 Runner 创建一个独立的文件夹。  
由于 `docker compose` 默认使用文件夹名字作为 project 的名字，所以每个 Gitlab Runner 最好使用不同的名字(即使它们在不同的父目录中)：
```
mkdir -p gitlab-runners-test
cp docker-compose.yaml gitlab-runners-test
cd gitlab-runners-test
```

由于 `docker-compose.yaml` 中需要从环境变量中读取一些配置，为了后续方便，可以把这些配置落到 `env.sh` 中：
```bash
export DOCKER_SOCKS=/var/run/docker.sock  
export HELM_CONFIG=$HOME/.config/helm
export DOCKER_CONFIG=$HOME/.docker
export KUBECTL_CONFIG=$HOME/.kube/config
```
然后再执行以下命令启动服务：
```shell
docker compose up
```
启动后，如果一切正常，会看到 `docker`, `docker compose`, `kubectl` 和 `helm` 的自检输出。
这时候，就可以进入容器里去注册 Gitlab Runner 了：

```
docker compose exec -it runner /bin/bash
```

进入容器后，执行：
```
# inside container
sudo gitlab-runner register
```
之后的操作，和常规的注册流程一样，唯一需要注意的是，选用 **`shell`** 作为 Gitlab Executor。

如果一切正常，退出刚才通过 `docker compose up `启动的服务，改为在后台常驻运行：
```bash
docker compose up -d
```
