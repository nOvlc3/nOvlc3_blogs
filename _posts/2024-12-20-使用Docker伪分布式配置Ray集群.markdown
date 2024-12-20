---
layout: post
title:  "使用Docker伪分布式配置Ray集群"
date:   2024-12-17 21:15 +0800
last_updated: 2024-12-18 09:15 +0800
categories: jekyll update
---

本文记述了笔者使用 Docker 启动了两个带有 Ray 的镜像实例，通过配置网络，实现了 Ray 集群的伪分布式配置。
> TODO：这是第一版，按照 Copilot 的推荐，还可以使用 docker-compose 来管理多个容器实例，以及使用 docker network 来管理容器之间的网络。

首先，我们需要准备一个有 docker 的服务器。然后，需要制作一个带有 Ray 的镜像。这里我使用手动的方法制作，更好的做法是使用 Dockerfile 来制作镜像。

我在拉取了基本的 Ubuntu 镜像后，通过 `docker run -it ubuntu:latest /bin/bash` 进入容器实例，然后执行下列操作:

- 换源(中科大源或清华源)
- 下载 miniconda3 并安装
- 使用 conda 配置一个名叫 ray 的虚拟环境并安装 ray 组件
- 使用 `docker ps | grep ubuntu` 查看容器的 id，然后使用 `docker commit -m "ray" -a "ray" 0b9e7f7f7f6d ray:latest` 提交镜像

这里我在几次安装后, 镜像通过 commit, 命名为 'ray_env:v3.0'。接着运行 `docker run -d --name ray_head -p 6380:6380 -p 8265:8265 ray_env:v3.0 tail -f /dev/null` 启动我们的 'ray_env' 容器实例, 并命名为 'ray_head', 同时, 映射了 6380 和 8265 端口。命令末尾的 `tail -f /dev/null` 是为了让容器保持运行状态。
> 我的几次安装, 主要是:
>
> 1. 安装 miniconda3
> 2. 在配置文件中写 `conda activate ray`, 方便每次进入容器实例后, 直接使用 `conda activate ray` 启动环境
> 3. 安装网络组件以查看 ip 地址

接着, 使用 `docker exec -it ray_head /bin/bash` 进入容器实例, 首先使用 `conda activate ray` 启动环境, 然后运行 `ray start --head --port=6380` 启动 Ray 集群的 head 节点。

同理, 我们可以使用 `docker run -d --name ray_worker1 ray_env:v3.0 tail -f /dev/null` 启动一个 worker 节点, 并使用 `docker exec -it ray_worker1 /bin/bash` 进入容器实例, 启动 worker 节点。在启动 conda 环境后, 运行 `ray start --address=<head 节点的 ip>:6380` 启动 worker 节点。这里要写入 ray_head 节点的 ip 地址。

回到 ray_head 节点, 运行 `python -c "import ray;ray.init(address='auto');print(ray.nodes())"` 查看节点信息, 可以看到有两个 'NodeID'。在在命令行中输入 `ray status` 可以看到有两个 Active 的 node。

至此，笔者通过粗略的、粗放的、无法复制、无法自动化、无法拓展的方式实现了第一个 Ray 集群的建立，这种方式介于完全分布式和伪分布式之间，可以用来学习 Ray 的基本操作。

TODO:

- [ ] 使用 Dockerfile 制作镜像
- [ ] 使用 docker-compose 管理多个容器实例
- [ ] 使用 docker network 管理容器之间的网络
