# Docker



**Docker的思想**

> 1. 集装箱
>
>    会将软件及运行环境一起打包成一个集装箱（镜像），发送到公共仓库，方便其他人直接拖拽。
>
> 2. 标准化
>
>    - 标准化的运输：docker可以从公共仓库拖拽某个打包好的集装箱。
>    - 命令的标准化：docker提供了一些列的命令去上传或下载集装箱
>    - 提供了REST的API
>
> 3. 隔离性
>
>    docker在运行集装箱的内容（容器）时，在Linux的内核中单独开辟一片空间，使得多个用户不会互相影响。



## 一、Docker的基本操作

### 2.1 安装Docker

### 2.2 Docker的中央仓库

> https://hub.docker.com/
>
> https://id.163yun.com/
>
> http://hub.daocloud.io/



### 2.3 镜像的操作

```sh
#拉取镜像到本地
docker pull 镜像名称[:tag]
```

---

```sh
#查看全部本地镜像
docker images
```

---

```sh
#删除本地镜像
docker rmi 镜像的标识
```

---

```sh
#镜像的导出
docker save -o 导出的路径 镜像的ID
#镜像的加载
docker load -i 镜像文件
#修改镜像名称
docker tag 镜像ID　新镜像的名称：版本
```



### 2.4 容器的操作

```sh
#运行容器
#简单操作
docker run 镜像的ID|镜像名称[:tag]
#常用操作
docker run -d -p 宿主主机端口:容器端口 --name 容器名称 镜像的ID|镜像名称[:tag]
# -d: 代表后太运行容器
# -p 宿主主机端口:容器端口:为了映射当前Linux的端口和容器的端口
# --name:指定容器的名称
```

---

```sh
#查看正在运行的容器
docker ps [-qa]
# -a: 查看全部的容器,包括没有运行的
# -q: 只查看容器的标识
```

---

```sh
#查看容器的日志
docker logs -f 容器ID
#-f: 滚动查看日志的最后几行
```

---

```sh
#进入到容器内部
docker exec -it 容器ID bash
```

---

```sh
#删除容器(需要先停止容器)
docker stop 容器ID
docker stop $(docker ps -qa)
docker rm 容器ID
docker rm $(docker ps -qa)#删除全部容器
```

---

```sh
#启动容器
docker start 容器ID
```