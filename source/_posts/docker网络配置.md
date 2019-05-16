---
title: docker网络配置
date: 2019-05-16 18:43:08
tags: [网络,docker]
categories:
- docker
comments: true
---

## 4种网络模式
docker支持4种网络模式
* **none** 容器除了本地接口'lo'，没有其他网络接口
* **host** 容器的网络配置与宿主机完全相同
* **bridge** 容器的网络接口连接到宿主机建立的bridge上

默认情况下，docker本来会创建3个网络：bridge、none和host，通过以下命令可查看docker中所有网络
```cmd
[root@localhost install]# docker network lsdocker network ls
NETWORK ID          NAME                DRIVER              SCOPE
0f4a6eea9183        bridge              bridge              local
8a9064b404c3        host                host                local
b00d9a442061        none                null                local
```

## 指定容器的网络模式
通过`docker run`命令从镜像创建容器时可通过`--net`参数指定网络模式
```cmd
docker run --net=none image_test
docker run --net=host image_test
docker run --net=bridge image_test
```

如果不指定网络模式，默认是bridge模式，docker默认会在宿主机上创建一个名为docker0的bridge，通过`ifconfig`可看到
```cmd
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:80ff:fea1:9eca  prefixlen 64  scopeid 0x20<link>
        ether 02:42:80:a1:9e:ca  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 16  bytes 1970 (1.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
该bridge的地址可通过配置文件修改，默认创建的镜像会将其网络接口绑定到docker0上

## 创建自定义docker网络
一般不使用docker0绑定容器，而是自定义网络
```cmd
docker network create --driver bridge net_name
```
然后将容器加入网络
```
docker run --net=net_name ...
```

## 容器访问外网
默认情况下，容器可访问外网，其原理是在宿主机上做了NAT转换，`iptables -t nat --nvL`查看iptables的nat表
```cmd
Chain PREROUTING (policy ACCEPT 1455 packets, 213K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    5   324 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 1131 packets, 192K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 17 packets, 1703 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 17 packets, 1703 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   73  4596 MASQUERADE  all  --  *      !br-0e12e16b15a1  172.19.0.0/16        0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
  365 23169 MASQUERADE  all  --  *      !br-ba1cdf022332  172.18.0.0/16        0.0.0.0/0           
    2   321 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255     
    0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24    

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  br-0e12e16b15a1 *       0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
    2   168 RETURN     all  --  br-ba1cdf022332 *       0.0.0.0/0            0.0.0.0/0
```
可以看到在POSTROUTING链上对源地址是172.17.0.0/16的数据流做了MASQUERADE操作，因此如果要禁用容器访问外网，则将其DROP掉
```cmd
iptables -t nat -I POSTROUTING -s 172.18.0.0/16 -j RETURN
```
或者使用filter表的FORWARD链，直接禁止其转发
```cmd
iptables -I FORWARD -s 172.18.0.0/16 -j DROP
```
