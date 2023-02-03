> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/whatday/article/details/122359836)

**目录**

[错误现象](#t0)

[错误原因](#t1)

[解决方法](#t2)

[方法一 删除没使用的网络](#t3)

[方法二 指定网络配置](#t4)

[方法三 修改 docker 默认网络地址（推荐）](#t5)

错误现象
----

[docker-compose](https://so.csdn.net/so/search?q=docker-compose&spm=1001.2101.3001.7020) up 报错

> Error response from daemon: could not find an available, non-overlapping IPv4 address pool among the defaults to assign to the network

错误原因
----

Docker 默认支持 30 个不同的自定义 bridge 网络，如果超过这个限制，就会提示上面的错误

使用命令 docker network ls 来查看创建的网络

```
[root@localhost test]# docker network ls
NETWORK ID     NAME			    DRIVER    SCOPE
d71725ca2226   bridge           bridge    local
549b36ab792b   host			    host      local
7512afa6db4c   none             null      local
9024b26834fd   test01			bridge    local
6a9389e3058f   test02			bridge    local
e1cf1dedb7dd   test03			bridge    local
fc4019ef418e   test04			bridge    local
29bba0bca81e   test05			bridge    local
7ee347eaec24   test06			bridge    local
28549a69cacc   test07			bridge    local
8df294348427   test08			bridge    local
5e56daa56894   test09			bridge    local
449033d11c39   test10			bridge    local
4eddaed16f58   test11			bridge    local
e02122c9c36a   test12			bridge    local
48f46a4c6bfc   test13			bridge    local
65f91ee63703   test14			bridge    local
d2a658f5ac03   test15			bridge    local
53b3335b2d7f   test16			bridge    local
4961f7af4876   test17			bridge    local
00074e196c25   test18			bridge    local
9853365387df   test19			bridge    local
7be93ba5d2df   test20			bridge    local
3890e77d93e3   test21			bridge    local
e711e49921d0   test22			bridge    local
754b16a98389   test23			bridge    local
0eec1dec9cc3   test24			bridge    local
80f95c508610   test25			bridge    local
d74984530daf   test26			bridge    local
ec21fcf6220c   test27			bridge    local
8b96b1d5a111   test28			bridge    local
1541b84f1724   test29			bridge    local
```

其中 bridge、host、none，是 docker 默认网络 且不能删除

统计数量：

```
[root@localhost test]# docker network ls | wc -l
33
```

除去 标题栏 和 默认的 host、none 正好有 30 个 bridge 网络，印证了 “Docker 默认支持 30 个不同的自定义 bridge 网络”

解决方法
----

### 方法一 删除没使用的网络

```
docker network prune

```

这个方法比较快速的 临时解决问题

### 方法二 指定[网络配置](https://so.csdn.net/so/search?q=%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE&spm=1001.2101.3001.7020)

修改 docker-compose.yml 文件

```
networks:
  default:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.1.0/24
```

或者 先创建网络 再指定使用这个网络配置

```
docker network create docker_compose_network --subnet 172.20.1.0/24

```

```
networks:
  default:
    external: 
      name: docker_compose_network 
```

此方法 需要修改 docker-compose.yml 文件 如果 docker-compose.yml 文件较多 会比较麻烦

### 方法三 修改 docker 默认[网络地址](https://so.csdn.net/so/search?q=%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80&spm=1001.2101.3001.7020)（推荐）

在 /etc/docker/daemon.json 追加

```
{
    ...
    "default-address-pools":[
        {"base":"172.20.0.0/16","size":24},
        {"base":"172.21.0.0/16","size":24},
        {"base":"172.22.0.0/16","size":24},
        {"base":"172.23.0.0/16","size":24}
    ]
}
```

注意 配置中的 “...” 是配置原本的其他内容，如果没有 /etc/docker/daemon.json 文件则新建 加入 default-address-pools 的配置即可

这个配置将允许 Docker 分配

172.20.[0-255].0/24

172.21.[0-255].0/24

172.22.[0-255].0/24

172.23.[0-255].0/24

每个网络允许访问 256 个地址，总共 1024 个网络。

加入后需要 删除现有网络占用

```
docker network prune  

```

然后重启 docker 服务

```
service docker restart

```

此时再使用 docker-compose up 创建 若干容器后 

使用 docker network ls 查看

```
[root@localhost test]# docker network ls
NETWORK ID     NAME			    DRIVER    SCOPE
d71725ca2226   bridge           bridge    local
549b36ab792b   host			    host      local
7512afa6db4c   none             null      local
9024b26834fd   test01			bridge    local
6a9389e3058f   test02			bridge    local
e1cf1dedb7dd   test03			bridge    local
fc4019ef418e   test04			bridge    local
29bba0bca81e   test05			bridge    local
7ee347eaec24   test06			bridge    local
28549a69cacc   test07			bridge    local
8df294348427   test08			bridge    local
5e56daa56894   test09			bridge    local
449033d11c39   test10			bridge    local
4eddaed16f58   test11			bridge    local
e02122c9c36a   test12			bridge    local
48f46a4c6bfc   test13			bridge    local
65f91ee63703   test14			bridge    local
d2a658f5ac03   test15			bridge    local
53b3335b2d7f   test16			bridge    local
4961f7af4876   test17			bridge    local
00074e196c25   test18			bridge    local
9853365387df   test19			bridge    local
7be93ba5d2df   test20			bridge    local
3890e77d93e3   test21			bridge    local
e711e49921d0   test22			bridge    local
754b16a98389   test23			bridge    local
0eec1dec9cc3   test24			bridge    local
80f95c508610   test25			bridge    local
d74984530daf   test26			bridge    local
ec21fcf6220c   test27			bridge    local
8b96b1d5a111   test28			bridge    local
1541b84f1724   test29			bridge    local
77232005cdee   test30			bridge    local
84ade32cd9b3   test31			bridge    local
68f934d048c4   test32			bridge    local
a0e9b94bb66b   test33			bridge    local
91e274044951   test34			bridge    local
c556a4e8bb6b   test35			bridge    local
7779879c023d   test36           bridge    local
310740c631db   test37			bridge    local
675e5167446f   test38			bridge    local
```

再统计其数量

```
[root@localhost test]# docker network ls | wc -l
42
```

bridge 已经突破了 30 的默认限制，目前可以有 1024 个

此方法一劳永逸，不用修改 docker-compose.yml 文件 一次修改 docker 配置 永久生效 

​