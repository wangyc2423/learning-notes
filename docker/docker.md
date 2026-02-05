# Docker

使用本机的内核，隔离进程/文件/网络

## image

镜像

一个只读的文件系统快照

## container

容器

镜像的实例

## Registry

仓库

镜像存放的地方

## 基本指令

` docker pull ubuntu:22.04`：拉镜像

` docker images`：查看本地镜像

` docker rmi ubuntu:22.04`：删除镜像

` docker run [选项] ubuntu:22.04 [命令]`：运行容器

* -d 后台运行

* -it 交互式终端，后更命令：/bin/bash

* --name [name] 给容器起名字

* -v 数据挂载，-v /home/user/code:/workspace

* -e 环境变量，-e CUDA_VISIBLE_DEVICES=0

* --cpus=8 --memory=32g：不加就默认全能用

* --gpus all或者0、1、2或者'"device=0,1"'

* --rm 容器退出后，自动删除这个容器

` docker ps`：查看运行中的容器

` docker ps -a`：查看所有容器（包括已停止）

` docker exec -it 容器名 /bin/bash`：进入正在运行的容器

```bash
docker stop 容器名  #停止容器
docker start 容器名 #启动
docker rm 容器名   #删除
```

# Dockerfile

```dockerfile
#每一行代表一层，都会对应新的shell进程
FROM ubuntu:22.04 #选一个基础镜像

RUN apt-get update && apt-get install -y python3 #build环境，结果会被固化进该镜像

WORKDIR /workspace #指定后续指令的工作目录

COPY . /workspace #从宿主机拷贝文件到镜像，会被docker run -v的覆盖

ENV PATH=$CUDA_HOME/bin:$PATH #设置环境变量

ARG TORCH_VER=2.1.0
RUN pip install torch==$TORCH_VER #build时期的变量，可指定

ENTRYPOINT ["python", "train.py"] #指定容器的作用，在容器启动的时候，若指定了该
                                  #参数，那真正的命令就是ENTRYPOINT + CMD

CMD ["bash"] #默认启动命令，会被docker run 的命令覆盖

```



build镜像

docker build --build-arg ABC=1 -t name . 
