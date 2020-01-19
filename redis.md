# 集群搭建

### 创建Redis 容器
- 创建redis配置文件（redis-cluster.tmpl）

   - 在路径/docker/redis下创建一个文件夹redis-cluster,在路径/docker/redis/redis-cluster下创建一个文件redis-cluster.tmpl，并把以下内容复制过去。

	```bash
	##节点端口
	port ${PORT}
	##开启集群模式
	protected-mode no
	##cluster集群模式
	cluster-enabled yes
	##集群配置名
	cluster-config-file nodes.conf
	##超时时间
	cluster-node-timeout 5000
	##实际为各节点网卡分配ip  先用上网关ip代替
	cluster-announce-ip  172.16.207.255
	##节点映射端口
	cluster-announce-port ${PORT}
	##节点总线端口
	cluster-announce-bus-port 1${PORT}
	##持久化
	appendonly yes
	# requirepass foobared
	# 指定密码root
	requirepass root

	```
	`备注：此模版文件为集群节点通用文件  其中${PORT} 将读取命令行变量  ip则根据网卡分配ip进行替换  以保证节点配置文件除端口以及ip 全部一致。`

- 创建自定义network
	```
	docker network create redis-net
	```
	通过docker network ls 查看网卡信息
	
	`备注：创建redis-net虚拟网卡 目的是让docker容器能与宿主（centos7）桥接网络 并间接与外界连接`
- 查看redis-net虚拟网卡网关ip,得到一个网关ip，修改上面的cluster-announce-ip
	```
	docker network inspect redis-net | grep "Gateway"
	```
- 在/docker/redis/redis-cluster下生成conf和data目标，并生成配置信息
	```
	for port in `seq 7000 7005`; do \
	mkdir -p ./${port}/conf \
	&& PORT=${port} envsubst < ./redis-cluster.tmpl > ./${port}/conf/redis.conf \
	&& mkdir -p ./${port}/data; \
	done
	```

	`备注：共生成6个文件夹，从7000到7005，每个文件夹下包含data和conf文件夹，同时conf里面有redis.conf配置文件,集群至少要6台redis`
- 创建6个redis容器
	```
	for port in `seq 7000 7005`; do \
	docker run -d -ti -p ${port}:${port} -p 1${port}:1${port} \
	-v /docker/redis/redis-cluster/${port}/conf/redis.conf:/usr/local/etc/redis/redis.conf \
	-v /docker/redis/redis-cluster/${port}/data:/data \
	--restart always --name redis-${port} --net redis-net \
	--sysctl net.core.somaxconn=1024 redis redis-server /usr/local/etc/redis/redis.conf; \
	done
	```
	```
  	备注：
	命令译为  
	循环7010 - 7015  运行redis 容器
	docker  run            运行
	-d                          守护进程模式
	--restart always     保持容器启动
	--name redis-710* 容器起名
	--net redis-net    容器使用虚拟网卡
	-p                        指定宿主机器与容器端口映射 701*:701*
	-P                        指定宿主机与容器redis总线端口映射 1701*:1701*
	--privileged=true -v /docker/redis/redis-cluster/701*/conf/redis.conf:/usr/local/etc/redis/redis.conf
	付权将宿主701*节点文件挂载到容器/usr/local/etc/redis/redis.conf 文件中
	--privileged=true -v /docker/redis/redis-cluster/${port}/data:/data \
	付权将宿主701*/data目录挂载到容器/data目录中
	--sysctl net.core.somaxconn=1024 redis redis-server /usr/local/etc/redis/redis.conf;
	容器根据挂载的配置文件启动 redis服务端```
- 通过命令docker ps可查看刚刚生成的6个容器信息
- 查看容器分配ip,去修改对应的config并重启容器。
	```
	docker network inspect redis-net
	```
	- 修改宿主挂载目录配置文件
	```
	cluster-announce-ip 172.21.0.2
	```
### 启动集群

- 进入一个节点
- 使用Redis 5创建集群，redis-cli只需键入：
	```
	redis-cli --cluster create 172.21.0.2:7000 172.21.0.3:7001 172.21.0.4:7002 172.21.0.5:7003 172.21.0.6:7004 172.21.0.7:7005 --cluster-replicas 1
	```
- 连接客户端，查看主从信息
  ```
  redis-cli -c -p 7000
  
  info replication
  ```

