
docker-compose需要另外安装
```
sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.4.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
```

### docker-compose使用
- 配置实例
```
# yaml 配置实例
version: '3'
services:
  web:
    build: .
    ports:
   - "5000:5000"
    volumes:
   - .:/code
    - logvolume01:/var/log
    links:
   - redis
  redis:
    image: redis
volumes:
  logvolume01: {}
```

- 命令

```shell
# 基于docker-compose.yml启动管理的容器   会从远程拉取相应镜像并启动容器
# 启动
docker-compose up
# 后台执行该服务可以加上-d 参数：
docker-compose up -d  

# 关闭并删除容器    删除容器，镜像还在，从新up会重新启动容器
docker-compose down

#开启关闭重启已经存在的由docker-compose维护的容器
docker-compose start|stop|restart

# 查看由docker-compose管理的容器
docker-compose ps

# 查看日志
docker-compose logs -f

# 重新构建
docker-compose build
```
