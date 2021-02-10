### Docker 基本概念

```
作用：部署（打包镜像一键运行）、升级（镜像层共享叠加）
	 运维（统一且隔离的环境）、启动（基于宿主机内核）
组成：镜像、容器、仓库
```

### Docker 基本操作

#### Docker 安装

```bash
# 卸载旧版本
$ yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
$ sudo remove -rf /var/lib/docker

# 设置安装工具
$ yum install -y yum-utils
$ yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装 docker 社区版
$ yum install docker-ce docker-ce-cli containerd.io

# 启动服务
$ systemctl start docker
$ docker version
```

#### Docker 常用命令

+ **镜像命令**

  ```bash
  docker images # 本地镜像列表 
  docker search # 搜索Hub上镜像
  docker pull   # 拉取镜像
  docker rmi -f # 删除镜像
  docker build -f dockerfile -t imgName[:tag] # 构建镜像
  docker commit containerid -t imgName[:tag]  # 提交镜像 
  docker push   # 推送镜像
  ```

+ **容器命令**

  ```bash
  # 创建并启动容器
  docker run [可选参数] image [DockerFile CMD/ENTRYPOINT参数]
  		--name cname    容器名
  		-d				后台运行
  		-it /bin/bash	进入容器执行操作
  		-p 	宿主机端口:容器端口	  端口映射
  		-e  环境变量	  DockerFile ENV参数
  		-v  [宿主机目录:]容器目录  挂载数据卷到宿主机，匿名挂载时，宿主机目录是/var/lib/docker/volumes/xxxx/_data（容器的数据有生命周期，本地则是永久）
  		--volumes-from cname    同步容器名为cname的数据卷
  		--net bridge|none|host	使用的网络：bridge 桥接模式（默认），宿主机路由，成对 Veth-Pair；none不配置网络；host与主机共享网络
  		--link otherCname		域名配置Host，通过容器名进行IP连接
  		
  		# 创建自定义网络，让不同的集群相互隔离
  		docker network create [可选参数] networkName
                  --driver bridge				网络模式
                  --subnet 192.168.0.0/16		子网掩码
                  --gateway 192.168.0.1		网关路由
                  
  # 容器列表
  docker ps -aq
  # 进入容器
  docker exec -it container /bin/bash
  # 退出容器
  Ctrl + p + q
  # 删除容器
  docker rm -f $(docker ps -aq)   # 所有容器
  # 停止和启动容器
  docker stop
  docker start
  docker restart
  # 文件手动拷贝
  docker cp 容器id:容器路径 宿主机路径
  docker cp 宿主机路径 容器id:容器路径 
  # 查看容器日志
  docker logs
  # 查看容器进程
  docker top
  # 查看容器元数据
  docker inspect
  ```

#### DockerFile

```bash
FROM 		# 基础镜像，进一步往上层构建
MAINTAINER 	# 镜像所有者

WORKDIR 	# 容器工作目录，进入容器
ADD 		# 构建容器时，添加宿主机数据到容器
RUN 		# 构建容器时，运行的命令

ENV 		# 当前上下文环境变量，-e
VOLUME 		# 数据卷的挂载目录，-v
EXPOSE 		# 端口配置，-p

CMD 		# 启动容器时执行的命令，会被 docker run 替换
ENTRYPOINT 	# 启动容器时执行的命令，docker run 不会替换，而是追加命令
			# shell格式：apt-get install python3
			# exec格式：["apt-get", "install", "python3", "param..."]
```

```dockerfile
FROM registry.cn-hangzhou.aliyuncs.com/anylogic/alogic-sgw:latest
MAINTAINER anylogic
WORKDIR $ALOGIC_HOME
ENV KETTY_APP=ctcloudet-backend
ADD src/main/resources $ALOGIC_HOME/alogic-ketty/sgw
RUN mkdir -p $ALOGIC_HOME/alogic-webapps/$KETTY_APP \
&& mkdir -p $ALOGIC_HOME/alogic-webapps/$KETTY_APP-gray \
&& ln -s $ALOGIC_HOME/alogic-ketty/apps/alogic-sgw-open $ALOGIC_HOME/alogic-webapps/$KETTY_APP/1.0 \
&& ln -s $ALOGIC_HOME/alogic-ketty/apps/alogic-sgw-open $ALOGIC_HOME/alogic-webapps/$KETTY_APP-gray/1.0

CMD ketty-web.sh start app=$KETTY_APP port=9800 daemon=false version=1.0
```

