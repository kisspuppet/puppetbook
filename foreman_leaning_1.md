#### foreman架构的引入1-foreman作为自动化运维工具为什么会如此强大


在引入foreman之前，笔者曾经大幅度测试过puppet的另外一个生态圈前端软件，那就是KermIT（[kermit.fr](http://kermit.fr)需要墙）。说实话基于KermIT这套架构还是相当不错的，尤其是在于mcollective的各种插件结合上做的很完美，可惜社区太不活跃，软件版本更新超慢，坑超多，最终还是放弃了。不过，他的架构还是值得借鉴的，对于那些想自己在puppet前端做UI的朋友可以多参考参考。

本文引入另外一个非常出色的前端管理工具Foreman，什么是foreman呢，官方是这样定义的：Foreman是一个物理和虚拟服务器的完整的生命周期管理工具(Foreman is a complete lifecycle management tool for physical and virtual servers)。

<!--more-->

**为什么要引入foreman作为配置管理工具的前端呢？**

本文从以下几个方面入手进行剖析

## 1、foreman的架构 ##

A Foreman installation will always contain a central foreman instance that is responsible for providing the Web based GUI, node configurations, initial host configuration files, etc. However, if the foreman installation supports unattended installations then other operations need to be performed to fully automate this process. The smart proxy manages remote services and is generally installed with all Foreman installations to allow for TFTP, DHCP, DNS, and Puppet, and the Puppet CA.

以上为官方的定义，我这里在根据日常使用的情况进行一些概括（以目前最新稳定版本1.5.3为例进行说明）

1. foreman本身只是一个框架，通过smart-proxy代理各种应用程序完成各项功能。
![Foreman框架](http://kisspuppet.com/img/foreman_architecture.png)

2. foreman通过代理DNS、DHCP、TFTP完成了kickstart、cobbler、jumpstart等各种自动化安装系统工具的图形统一管理窗口，实现的结果是只需要在foreman上定制各种模板（pxe、ks），不同的模板还可以嵌套各种片段（snippet）达到统一、简化的目的。完成之后，便可以添加节点，关联定义的各种模板生成各种的pxe和ks文件实现自动化安装。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_2.png)
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_3.png)

3. foreman通过代理puppet、puppet CA完成对puppet自动签名、puppet环境、class、变量、facter的管理。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_4.png)
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_5.png)

4. foreman通过ENC和静态组管理class和node之间的关联。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_6.png)

5. foreman通过puppet plugin，可以在UI上完成对节点puppet命令的触发动作，触发的方法可以借助puppetkick（已经被遗弃）、mcollective（借助sudo）、puppetssh（借助sshkey）、salt、customrun等各种工具实现。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_7.png)

6. foreman可以收集所有节点运行puppet后的报告、执行情况。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_8.png)

7. foreman还提供了各种搜索、报表等功能，能够更好的展现节点的运行状况。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_9.png)

8. foreman除了管理裸机外还可以管理各种虚拟化软件，比如RHEV-M、EC2、VMWware和OpenStack等。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_10.png)

9. foreman还可以和LDAP以及AD集成。
10. foreman还提供了强大了用户、权限管理入口，可以建立多个用户、多个用户组、还可以对权限进行角色的定义等。不同的权限用户在UI上所看到的功能以及主机是不一样的。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_11.png)

11. foreman还提供了所有在UI上操作的Audits（审计）功能，这样可以保障所有用户的操作都有据可查。
![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_12.png)

除此之外，还有其它很多功能。。。。

**针对配置管理的不足之处：foreman和mcollective的结合并不是很好，它仅仅是借用了puppetkick的插件集成了mcollective的一条命令而已，这方面后期是否会有改进还需要等待。**

## 2、foreman的版本蓝图 ##

**以下为foreman的版本发展线路图**

![Foreman版本发展线路图](http://kisspuppet.com/img/foreman_leaning_1.png)

从图中可以看出，foreman的发展是相当的迅速的，无论是版本更替上还是社区的活跃度上都是相当的良好。目前最新稳定版本为**1.5.3**（统计时间2014年10月18号）。

**版本目前发展和预期线路图：**[http://projects.theforeman.org/rb/releases/foreman](http://projects.theforeman.org/rb/releases/foreman)

## 3、foreman的社区活跃度 ##

**foreman google groups：**

[https://groups.google.com/forum/#!forum/foreman-users](https://groups.google.com/forum/#!forum/foreman-users)

[https://groups.google.com/forum/#!forum/foreman-dev](https://groups.google.com/forum/#!forum/foreman-dev)

**foreman的IRC:**
"#theforeman"

[http://webchat.freenode.net/](http://webchat.freenode.net/)


