# arplab

吉林大学ARP欺骗攻击演示平台

## 实验原理

ARP(地址解析协议)，在一个局域网内，主机A(IP_a,MAC_a)和主机B(IP_b,MAC_b)通信需要知道对方的的MAC地址，但是由于应用层的协议只包含了目的ip地址和源ip地址，并没有mac地址的信息。通过ip地址查询mac地址，需要使用arp。每台主机内部都有arp缓存用于保存同一个网络中其他机器的mac地址和ip地址，于是发包前先根据目的ip地址在缓存arp表中查找相应的表项，如果可以查到那么可以通过以太网传输。如果没有对应的表项，那么主机会在局域网中发起arp request广播期望得到目的主机的一个arp reply响应包，并将响应包中的mac地址填入到arp缓存表中。

**arp缓存更新机制**

每一台主机都会收到arp reply并根据响应包中的ip和mac检查自己的arp缓存表，如果不一致(或者不存在)那么会更新arp缓存表

**攻击方式**

主机A(IP_a,MAC_a)和主机B(IP_b,MAC_b)通信，正常情况下主机A发出arp请求询问IP_b的mac地址,主机B会发出一个arp reply更新主机A的arp缓存表为(IP_b,MAC_b)，同样的主机B会询问IP_a的mac地址，并更新arp缓存表(IP_a,MAC_a)。如果存在一个中间人主机M(IP_m,MAC_m)，不断的发出arp reply(IP_a,MAC_m)和(IP_b,MAC_m)，由于arp的更新机制，主机A和B中的对应的mac地址会被修改为MAC_m,那么主机A,B都会将数据发送个主机M，如果主机M具有转发机制，可以实现监听A,B间通话的效果。

## 实验环境

**成员组成**:

- 被攻击者(tcpdumper和victim)

- 攻击者(arpspoofer)

### 实验流程

**启动容器**

采用docker-compose一键启动 `docker-compose up -d`

![image-20210518184700140](/home/badmonkey/code/ctf/web/lab-arpspoof/arplab/img-1.png)

打开三个终端一次进入对应的容器中

`docker exec -it container-name shell` 例:`docker exec -it arplab_victim_1 /bin/sh`

**获取ip信息**

分别查看ip信息`ip addr`

![image-20210518185008497](/home/badmonkey/code/ctf/web/lab-arpspoof/arplab/img-2.png)

得到三个容器对应的ip例:

```text
victim ==> 172.26.0.2
tcpdumper ==> 172.26.0.4
arpspoofer ==> 172.26.0.3
gateway == > 172.26.0.1
```

首先查看victim的arp缓存，如果不为空，则将其清空

`arp -a`查看所有的缓存信息

`arp -d ip`删除对应ip项的缓存信息

**开启监听并建立通信**

首先开启开启tcpdumper的监听

`tcpdump -n -i eth0`

victim容器ping tcpdumper 以建立连接，可以收到类似下面的包

![image-20210518190011753](/home/badmonkey/code/ctf/web/lab-arpspoof/arplab/img-3.png)

由于victim的arp表是空的，所以第一个请求会发起arp request请求tcpdumper的mac地址，收到arp reply后发出icmp包，同样的会更新tcpdumper中victim的mac信息。

**开启arp欺骗**

首先将victim中的tcpdumper的arp缓存删掉(为了演示效果更好)

然后使用arpspoof进行arp泛洪攻击， `arpspoof -i eth0 -t 172.26.0.2 172.26.0.4` (污染172.26.0.4)

`arpspoof -i eth0 -t 172.26.0.4 172.26.0.2`(污染172.26.0.2)

最后污染了victim的缓存表

![image-20210518191319967](/home/badmonkey/code/ctf/web/lab-arpspoof/arplab/img-4.png)

## 参考链接

https://dockersec.blogspot.com/2017/01/arp-spoofing-docker-containers_26.html

https://github.com/BrunoVernay/lab-arpspoof