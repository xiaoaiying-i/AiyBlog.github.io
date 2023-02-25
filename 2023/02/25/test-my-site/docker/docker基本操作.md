### 构建镜像推送到仓库
-  docker engine添加仓库地址后重启
-  登录 
docker login aaa.com:80 -u "robot$admin" -p 
-  添加一个tag
docker tag  etcd-icsl:1.0.0  test.docker.com:80/auth/etcd-icsl:1.0.0
- push到仓库
docker push test.docker.com:80/auth/etcd-icsl:1.0.0

```
# 构建etcd-icsl
docker build -t test.docker.com:80/auth/etcd-icsl:1.0.3 .
# 构建app
docker build --no-cache=true -t test.docker.com:80/auth/app:1.2.1 .
```

### docker 镜像导出导入
```
docker save -o euler1.0.1.tar 8d83b93
docker load --input ./euler1.0.1.tar
```

### 远程仓库操作

- https://www.jianshu.com/p/3ef44bcfd177

```

```

### docker命令
```
# 保存镜像 
docker save -o path/.../name.vertion.tar dockerId
# 加载镜像
docker load -i path/.../name.vertion.tar
# 添加tag
docker tag dockerId  dockerName:version
# 查看运行镜像
docker ps
# 进入容器
docker exec -it dokcerId /bin/bash

# 删除所有已经停止的容器  注意：要先确认停止的容器中是否有不可以删除的，也可以删除后使用镜像再启一个容器。
docker rm $(docker ps -a|grep Exited |awk '{print $1}')docker rm $(docker ps -qf status=exited)

# 删除所有未打标签的镜像
docker rmi $(docker images -q -f dangling=true)

# 删除所有无用的volume
docker volume rm $(docker volume ls -qf dangling=true)

# 拉取镜像
docker pull test.docker.com:80/auth/etcdmirror:1.0.2
```