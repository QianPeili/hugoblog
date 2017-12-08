---
title: "用 Docker搭建 MongoDB replica set"
date: 2017-12-08T10:46:55+08:00
categories:
  - action
tags:
  - docker
  - mongodb
---

昨天 `Mongodb 3.6` 发布了，想尝试下新特性，尤其是 `Change stream`，写完代码跑起来，结果告诉我需要 `replica set` 模式下才能使用，坑爹啊！于是临时在网上找了找相关资料，准备用 `docker` 搭一个简单的 `replica set`（下文统一用“副本集”称呼）。

## 使用 `Docker` 搭建副本集

本段内容主要参考了网上 **Soham Kamani** 写的 [Creating a MongoDB replica set using Docker 🍃](https://www.sohamkamani.com/blog/2016/06/30/docker-mongo-replica-set/)。更详细的内容可以查看原文。


##### 安装docker-ce

[Get Docker CE for Ubuntu | Docker Documentation](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-docker-ce-1)

##### 下载 mongo:3.6 镜像

	docker pull mongo:3.6

##### 创建副本集专用 Network
	
	docker network create mongo-cluster

##### 启动 3 个 mongo 实例

	docker run -P -d \
	--name mongo1 \
	--net mongo-cluster \
	mongo:3.6 mongod --bind_ip 0.0.0.0 --replSet mongo-set

	docker run -P -d \
	--name mongo2 \
	--net mongo-cluster \
	mongo:3.6 mongod --bind_ip 0.0.0.0 --replSet mongo-set

	docker run -P -d \
	--name mongo3 \
	--net mongo-cluster \
	mongo:3.6 mongod --bind_ip 0.0.0.0 --replSet mongo-set

其实3个命令除了名字没有其他不同，`-P` 的作用是让系统自己选择一个大端口号对镜像暴露的端口进行映射，懒人必备。

这里跟参考问答有个不同是设置了 `--bind_ip 0.0.0.0`， 因为至少在 mongo:3.6 的镜像里，默认监听的是本地地址，导致容器建无法互相连接，无法进行副本集设置。

##### 初始化副本集设置

通过 docker 命令进入 mongo1 的数据库交互界面

	docker exec -it mongo1 mongo

然后用开始配置副本集配置

	MongoDB shell version v3.6.0
	connecting to: mongodb://127.0.0.1:27017
	MongoDB server version: 3.6.0
	Welcome to the MongoDB shell.
	> config={"_id":"mongo-set","members":[{"_id":0,"host":"mongo1:27017"},{"_id":1,"host":"mongo2:27017"},{"_id":2,"host":"mongo3:27017"}]}
	{
	  	"_id" : "mongo-set",
	  	"members" : [
	  		{
	  			"_id" : 0,
	  			"host" : "mongo1:27017"
	  		},
	  		{
	  			"_id" : 1,
	  			"host" : "mongo2:27017"
	  		},
	  		{
	  			"_id" : 2,
	  			"host" : "mongo3:27017"
	  		}
	  	]
	  }
	> rs.initiate(config)
	{
	        "ok" : 1,
	        "operationTime" : Timestamp(1512702787, 1),
	        "$clusterTime" : {
	                "clusterTime" : Timestamp(1512702787, 1),
	                "signature" : {
	                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
	                        "keyId" : NumberLong(0)
	                }
	        }
	}

看到 `"ok": 1`，副本集就配好了，通过 `rs.status()` 可以查看副本集状态。而我的命令提示符已经变成了： `mongo-set:SECONDARY> `。（其实比较奇怪，`mongo1` 应该是 `PRIMARY` 才对）


### 使用 `Docker-compose` 搭建副本集

上面的方式，虽然比手动在宿主机上搭个副本集方便太多了，但是还是有点麻烦，所以我准备尝试用 `docker-compose` 更快速地搭建一个副本集。

##### 安装 docker-compose

[Install Docker Compose | Docker Documentation](https://docs.docker.com/compose/install/)

##### 编写 docker-compose.yml

	version: '3'
	services:
	  mongo4:
	    image: mongo:3.6
	    networks:
	      - mongo-cluster-2
	    command: mongod --bind_ip 0.0.0.0 --replSet mongo-set-2
	    
	  mongo5:
	    image: mongo:3.6
	    networks:
	      - mongo-cluster-2    
	    command: mongod --bind_ip 0.0.0.0 --replSet mongo-set-2
	    
	  mongo6:
	    image: mongo:3.6    
	    networks:
	      - mongo-cluster-2
	    command: mongod --bind_ip 0.0.0.0 --replSet mongo-set-2
	    
	networks:
	    mongo-cluster-2:

##### 启动服务

	docker-compose up -d

`-d` 参数表示以 `daemon` 形式运行服务。用 `docker ps` 命令可以看到新的3个容器已经起来了。

	CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                 PORTS                                               NAMES
	a973bacfecb6        mongo:3.6           "docker-entrypoint..."   5 seconds ago       Up 3 seconds           27017/tcp                                           mongoreplcompose_mongo5_1
	f98b7ef74a47        mongo:3.6           "docker-entrypoint..."   5 seconds ago       Up 4 seconds           27017/tcp                                           mongoreplcompose_mongo6_1
	58e7ecb2c474        mongo:3.6           "docker-entrypoint..."   5 seconds ago       Up 4 seconds           27017/tcp                                           mongoreplcompose_mongo4_1

##### 配置副本集

后续操作就跟之前一样了，不再赘述。


### 总结

有 `docker` 还是很方便地，但还是只会简单地使用。这次搭建副本集，也是用了最简单的参数，不适合用在生产环境。不过作为开发环境的搭建，还是绰绰有余的～～