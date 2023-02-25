

### docker 网络隔离

https://blog.51cto.com/u_14156658/2495935


```
1、Docker网络隔离原理
需要管理创建网络空间名称；将不同的容器加载到不同的网络空间名称中实现隔离；默认不配置网络隔离默认给容器分配的docker0网络空间名称。

2、Docker容器自带的网络空间名称类型
- bridge：容器桥接到docker0网桥上；
- host：容器同步docker宿主机的网络配置信息；
- none：不创建网络，docker容器不需要配置TCP/IP信息；

3、配置Docker网络名称空间隔离

<!--查看docker默认的网络名称空间-->
[root@centos01 ~]# docker network ls   
NETWORK ID          NAME                DRIVER              SCOPE
8bb953004416        bridge              bridge              local
2c18234cad82        host                host                local
67860e823c36        none                null                local

<!--创建网络名称空间-->
[root@centos01 ~]# docker network create -d bridge liyanxin  
0c69de4672ec173dc4c60b19e0bf93b361f45a804859f7bc2105d85ca83b1169

<!--创建网络名称空间-->
[root@centos01 ~]# docker network create -d bridge gongsunli   
35687468c9034262173a96e9c23e045cbb8b7ffa6648fc84e015504740815001

<!--查看docker宿主机网卡信息-->
[root@centos01 ~]# ifconfig   
br-0c69de4672ec: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 0.0.0.0

br-35687468c903: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.19.0.1  netmask 255.255.0.0  broadcast 0.0.0.0

 <!--创建运行的容器添加到liyanxin网络名称空间中隔离-->
[root@centos01 ~]# docker run -it -d --name centos6.701 --network=liyanxin hub.c.163.com/public/centos:6.7-tools    
b85a2d8419a98756369ddc3b78247d3d42c178e8e563a936fe973f2f6611f951

<!--登录centos6.701容器--><!--查看IP地址-->
[root@centos01 ~]# docker exec -it centos6.701 /bin/bash   
[root@b85a2d8419a9 /]# ifconfig    
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:02  
          inet addr:172.18.0.2  Bcast:0.0.0.0  Mask:255.255.0.0

<!--创建运行的容器添加到gongsunli网络名称空间中隔离-->
[root@centos01 ~]# docker run -it -d --name centos6.702 --network=gongsunli hub.c.163.com/public/centos:6.7-tools    
9af0fb7b85af3270f3c7c44b62438f436b22289ac0a7604d6ed522604b7b185f
<!--登录centos6.702容器--><!--查看IP地址-->
[root@centos01 ~]# docker exec -it centos6.702 /bin/bash  
[root@9af0fb7b85af /]# ifconfig    
eth0      Link encap:Ethernet  HWaddr 02:42:AC:13:00:02  
          inet addr:172.19.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
```

- 进入命名空间
```
# 启动容器后，查看该容器的第一个进程pid
docker inspect -f {{.State.Pid}} 容器id

# 执行命令进入了命名空间
nsenter --target pid --type(net、ipc、PID等命名空间)，

# 退出
exit
```

