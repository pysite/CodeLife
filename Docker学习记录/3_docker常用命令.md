

> 官方文档：https://docs.docker.com/reference/

# Docker常用命令

## 帮助命令：

```shell
docker version
docker info
docker 命令 --help 	#万能help
```



## 镜像命令

**查看本地镜像**：

```shell
docker images
docker images --help

docker images -aq	#显示本地所有image的id
```

文档地址：https://docs.docker.com/engine/reference/commandline/images/

**搜索镜像**：（也可以直接去docker hub网站中搜索）

```shell
docker search 镜像名
docker search --help
```

**下载镜像**：

```shell
docker pull 镜像名
docker pull 镜像名:版本号
docker pull --help
```

**删除镜像**：

```shell
docker rmi -f 镜像id	#删除单个
docker rmi -f 镜像id 镜像id 镜像id	#删除多个
docker rmi -f $(docker images -aq)	#删除全部
```



## 容器命令

**先下载centos的镜像**：

```shell
docker pull centos
```

**新建容器并启动**：

```shell
docker run [可选参数] centos

#参数说明
--name="centos1" 容器名字来区分容器
-d				 后台方式运行
-it				 使用交互方式运行，进入容器查看内容
-p				 指定容器的端口
	-p ip:主机端口:容器端口
	-p 主机端口:容器端口
	-p 容器端口
	容器端口
-P				 随机指定端口

#举例
docker run -it centos /bin/bash	#会发现前面的主机名称变化
ls
exit	#从容器退回主机
ls
docker ps	#查看当前运行的容器
docker ps -a #查看曾经运行的容器

#docker ps 命令
		#列出当前正在运行的容器
-a 		#列出当前正在运行的容器+列出历史运行过的容器
-n=?	#显示最近创建的?个容器
-q		#只显示容器id
```

**退出容器**：

```shell
exit		#容器停止并退出
ctrl+P+Q	#容器不停止退出
```

**删除容器**：

```shell
docker rm 容器id
docker rm -f $(docker ps -aq)	#删除所有容器, -f是强制删除（包括正在运行的）
```

**启动和停止容器**：

```shell
docker start 	容器id	
docker restart 	容器id
docker stop 	容器id	#停止
docker kill 	容器id	#强制停止
```



## 常用其他命令

**后台启动容器**：

```shell
#命令 
docker run -d centos
#问题：docker ps发现centos停止了
#原因：docker容器使用后台运行，必须要有一个前台进程，否则会自动停止
```

**查看日志**：

```shell
docker logs
docker logs --help
```

**查看容器中进程信息**：

```shell
docker top 容器id
```

**查看镜像元数据**：

```shell
docker inspect 容器id
docker inspect --help
```

**进入当前正在运行的容器**：

```shell
#通常容器以后台方式运行，需要进入容器修改一些配置

#方式一
docker exec -it 容器id /bin/bash

#方式二
docker attach 容器id

#exec	进入容器后开启一个新的终端，可以在里面操作
#attach	进入容器正在执行的终端
```

**从容器内拷贝文件到主机上**：

```shell
docker cp 容器id:容器内路径 目的主机路径
```

容器被停止了但里面的数据还会在。

