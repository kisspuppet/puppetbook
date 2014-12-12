#### Puppet基础篇6-Puppet更新方式的选型


基于C/S架构的Puppet更新方式一般有两种，一种是Agent端设置同步时间主动去PuppetMaster端拉取配置，另一种是通过PuppetMaster端使用`puppet kick`命令或者借助mcollctive触发更新配置，两种方式适应不同的生产环境，各具特色。
<!--more-->

## 一、主动更新 ##

主动更新就是节点运行的puppet守护进程到预设的时候后自动去和puppetmaster进行交互直至更新完成的过程。
这种更新方式不易控制，主要表现在以下几个方面：

**优点：**

- 节点定期主动更新，无论是谁将节点被puppet管理的配置更改了，都会在规定的时间内自动修复，无须管理员登录查看。

- 环境搭建简单，不需要很复杂的架构，puppet本身C/S架构便可以完成。

...


**缺点：**

- 节点数量过大的情况下同时会向puppetmaster端发起更新请求，会造成puppetmaster性能瓶颈。当然，也有一些解决方案比如设置任务计划，节点分批进行更新。

- 由于节点会定期向puppetmaster端提取配置进行更新，这要求puppetmaster端的环境要足够的安全。否则，任何人上去修改puppet模板都会造成节点同步更新，如果有人写了可执行资源“rm -rf /”，那损失就大了。

- 不能手动控制那些服务器需要更新，那些不需要更新。

...


自动更新方式配置很简单的，只需要在节点配置文件puppet.conf添加runinterval字段即可实现自动更新，以下步骤简单测试下

**注：**默认情况下，puppet.conf配置文件中是没有runinterval字段的，如果不配置，默认是每隔30分钟自动同步一次。

**1、添加runinterval字段**

为了方便测试，可设置为10秒

	[root@agent1 ~]# vim /etc/puppet/puppet.conf
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com
	    certname = agent1_cert.kisspuppet.com
	    runinterval = 10   #设置同步的间隔时间，单位为秒

**2、启动puppetagent服务**

	[root@agent1 ~]# /etc/init.d/puppet start
	Starting puppet:                                           [  OK  ]

**3、打开message日志查看同步状态**
可以看到日志里面每隔10秒钟agent1向puppetmaster同步一次

	[root@agent1 ~]# tailf  /var/log/messages
	Mar 11 23:39:42 agent1 kernel: ide: failed opcode was: 0xec
	Mar 11 23:39:42 agent1 smartd[3110]: Device: /dev/hdc, not ATA, no IDENTIFY DEVICE Structure 
	Mar 11 23:39:42 agent1 smartd[3110]: Device: /dev/sda, opened 
	Mar 11 23:39:42 agent1 smartd[3110]: Device: /dev/sda, IE (SMART) not enabled, skip device Try 'smartctl -s on /dev/sda' to turn on SMART features 
	Mar 11 23:39:42 agent1 smartd[3110]: Monitoring 0 ATA and 0 SCSI devices 
	Mar 11 23:39:42 agent1 smartd[3112]: smartd has fork()ed into background mode. New PID=3112. 
	Mar 11 23:39:42 agent1 avahi-daemon[3076]: Server startup complete. Host name is agent1.local. Local service cookie is 773321440.
	Mar 11 23:44:11 agent1 puppet-agent[3210]: Reopening log files
	Mar 11 23:44:12 agent1 puppet-agent[3210]: Starting Puppet client version 2.7.25
	Mar 11 23:44:16 agent1 puppet-agent[3210]: Finished catalog run in 1.53 seconds
	Mar 11 23:44:29 agent1 puppet-agent[3210]: Finished catalog run in 0.96 seconds
	Mar 11 23:44:40 agent1 puppet-agent[3210]: Finished catalog run in 0.20 seconds
	Mar 11 23:44:51 agent1 puppet-agent[3210]: Finished catalog run in 0.24 seconds
	Mar 11 23:45:02 agent1 puppet-agent[3210]: Finished catalog run in 0.21 seconds
	Mar 11 23:45:13 agent1 puppet-agent[3210]: Finished catalog run in 0.22 seconds

**4、通过命令进行测试**

同样，每隔10秒会同步一次

	[root@agent1 ~]# /etc/init.d/puppet stop
	Stopping puppet:                                           [  OK  ]
	[root@agent1 ~]# puppet agent --verbose --no-daemonize 
	notice: Starting Puppet client version 2.7.25
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: Finished catalog run in 0.21 seconds
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: Finished catalog run in 0.20 seconds
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: Finished catalog run in 0.20 seconds
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: Finished catalog run in 0.21 seconds
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: Finished catalog run in 0.21 seconds


## 二、推送更新 ##

推送更新就是通过puppet kick或者mcollective来控制节点什么时候向puppetmaster端获取配置变更信息。这种方式比较容易控制，主要表现在以下几个方面：

**优点：**

- 非常容易控制节点的更新周期

- 安全性比较高，每次更新之前可先检查好代码后再更新

- 可以针对某一个cluster（一组服务器）进行推送更新，灵活性很强。

- 扩展性很强，可整合各种其他平台

...

**缺点：**

- 环境搭建比较复杂，需要部署N多东西

- agent端配置被篡改后，需要手动触发才能够恢复，不能够及时恢复

...


**1、puppet kick方式**

puppet kick是是通过puppetmaster端的命令触发的方式进行更新的，由于其锁的问题很难解决外加上主机单元控制不是很灵活，逐渐被抛弃了，puppetlabs也看到了这一点，因此收购了mcollecitve。当然，这种方式在很多企业中还在用，这里给几个大家方式参考：

[http://purplegrape.blog.51cto.com/1330104/1179358](http://purplegrape.blog.51cto.com/1330104/1179358)

[http://dreamfire.blog.51cto.com/418026/1279878](http://dreamfire.blog.51cto.com/418026/1279878)

**2、mcollective触发方式**
需要搭建mcollective+MQ架构，搭建好了之后通过mco命令将puppet命令传输至MQ分配到一组节点上去，关于mcollective+MQ架构搭建会在第三部分详细讲解，也可以参考之前写的一篇文章 [http://kisspuppet.com/2013/11/10/mcollective-middleware/](http://kisspuppet.com/2013/11/10/mcollective-middleware/)

	[root@webui ~]# mco puppet -v runonce 
	Discovering hosts using the mc method for 2 second(s) .... 3
	
	 * [ ============================================================> ] 3 / 3
	
	
	node3.rsyslog.org                       : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30' command"}
	
	node2.rsyslog.org                       : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30' command"}
	
	node1.rsyslog.org                       : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30' command"}
	
	
	
	---- rpc stats ----
	           Nodes: 3 / 3
	     Pass / Fail: 3 / 0
	      Start Time: Tue Mar 11 17:40:56 +0800 2014
	  Discovery Time: 2003.85ms
	      Agent Time: 1132.44ms
	      Total Time: 3136.29ms

**3、封装成WebUI的触发更新**

Kermit架构--一个非常完美的基于MCollective和Puppet，并结合Django和Rest组成的Web框架（第四部分会讲）
![kermit架构图](http://kisspuppet.com/img/kermit-MCollective.png)

搭建完成之后的操作界面

选中puppet插件
![kermit界面](http://kisspuppet.com/img/kermit-1.jpg)

执行puppet推送动作
![kermit界面](http://kisspuppet.com/img/kermit-2.jpg)

执行过程
![kermit界面](http://kisspuppet.com/img/kermit-3.jpg)

显示结果
![kermit界面](http://kisspuppet.com/img/kermit-4.jpg)



