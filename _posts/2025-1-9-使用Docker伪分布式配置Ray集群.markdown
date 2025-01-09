---
layout: post
title:  "使用 Docker 伪分布式配置 Ray 集群"
date:   2025-1-9 19:15 +0800
categories: jekyll update
---

## 简介

本文记述了笔者使用 Docker 启动了多个带有 Ray 的镜像实例，通过配置网络，实现了 Ray 集群的伪分布式配置。

如果有 kubernetes 这样的集群配置工具，应该更加容易实现接下来的操作，但笔者感觉接下来的做法看 kubernetes 有共通之处，就是 yaml 配置文件。kubernetes 中可以用 Helm 方便的管理 yanl 文件，下面要介绍的做法同样是使用 yaml。可惜笔者暂时对 yaml 格式的配置文件的历史和未来了解太少，无法讲述其优点。

## 准备

- 一台服务器
- 装有 docker

## 构建镜像

第一步，使用 `docker pull python:3.12` 命令拉取 python官方镜像, 或者建立一份 pythonfile, 通过 `docker build -f pythonfile -t ray_env:v1.0 .` 构建, pythonfile 内容如下：
> - `-t` 指定创建的镜像名称
> - `-f` 指定用于构建镜像的文件

```docker
# 基础镜像
FROM python:3.12

RUN pip install ray

# 默认的 python 镜像，通过 docker run 运行会直接进入 python 解释器
CMD ["/bin/bash"]  
```

运行结果：

![build](../imgs/ray_cluster/PixPin_2025-01-09_19-47-57.png)

> 之前笔者先拉取 ubuntn 镜像, 再安装 miniconda3 来创建 python 环境, 这种方式会有环境上的问题, 笔者用着不好

通过 `docker images | grep ray` 查看镜像是否创建成功

![ray](../imgs/ray_cluster/PixPin_2025-01-09_19-46-05.png)

## 编写 yaml 配置文件

这一部分通过 AI 辅助，笔者得到一份基本的 yaml 配置文件，如下：

```docker
version: '3.8'
services:
  head:
    image: ray_env:v1.0
    container_name: ray_head
    command: >
      bash -c "ray start --head --port=6380 --redis-password='5241590000000000' && tail -f /dev/null"
    ports:
      - "6380:6380"
      - "8265:8265"  # Dashboard port
    networks:
      ray_net:
        ipv4_address: 172.128.0.2
  worker:
    image: ray_env:v1.0
    depends_on:
      - head
    command: >
      bash -c "ray start --address='ray_head:6380' --redis-password='5241590000000000' && tail -f /dev/null"
    networks:
      ray_net:
    deploy:
      replicas: 3  # 启动3个worker实例

networks:
  ray_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.128.0.0/24
```

笔者对关键几点进行解释说明：

- 尽管这份 yaml 文件中笔者对网络进行了配置，但这不是必要的, `docker-compose` 命令似乎能自动配置 `/etc/hosts`这样的文件从而实现 'DNS 解析', 即将头节点, 这里命名为 'ray_head' 的节点进行解析, 获取其 ip 地址
- 更改 `replicas` 可以启动不同数量的 worker 实例
- `command` 一栏, 因为 `pythonfile`中编写了启动命令为 `CMD ["/bin/bash"]` 故可以直接使用 `ray start` 命令启动集群
- `ports` 用于端口转发, 但要注意不能与服务器已经使用的端口冲突, 可以通过 `lsos -i :<端口号>` 以及 `docker ps | grep <端口号>` 进行排查

### 启动结果

通过 `docker-compose -f ray_cluster.yml up -d` 启动 Ray 集群

![up](../imgs/ray_cluster/PixPin_2025-01-09_20-01-02.png)

通过 `docker exec -it ray_head /bin/bash` 进入头节点， 然后通过 `ray status` 检查集群情况:

![exec](../imgs/ray_cluster/PixPin_2025-01-09_20-01-52.png)


使用 `docker-compose -f ray_cluster.yml down` 停止集群运行

![down](../imgs/ray_cluster/PixPin_2025-01-09_19-58-51.png)
