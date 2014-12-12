#### Puppet基础篇9-Puppetmaster多环境配置


# 扩充现有架构环境是对一个企业成长的见证 #

将基础环境模块部署到puppetmaster端之后就可以初始化所有节点了，接下来就是部署应用代码了。众所周知，一个企业中应用代码的编写并不是运维一个人完成的，而且代码的上线也不是一次性完成的。标准的架构应该由开发、测试、生产三个组成，对应到puppetmaster里面应该有3套代码才对。而且每套代码都应该对应到自己的环境中，而代码的变更更应该通过版本控制工具进行管理，比如svn、git等。
接下来我们为puppetmaster创造3个环境，它们分别是开发环境(kissdev)、测试环境(kissqa)、生产环境(kissprd).
<!--more-->

## 1、配置puppet.conf ##

在标签[master]中添加environments环境，其次创建对应的环境标签及配置

	[root@puppetmaster ~]# vim /etc/puppet/puppet.conf
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com
	    certname = puppetmaster_cert.kisspuppet.com
	[master]
	    certname = puppetmaster.kisspuppet.com
	    environments = kissdev,kisstmq,kissprd  #添加三个环境的标签名称
	[kissdev]
	modulepath = $confdir/environments/kissdev/environment/modules:$confdir/environments/kissdev/application/modules  #设置环境的搜索路径
	manifest = $confdir/environments/kissdev/manifests/site.pp  #设置环境的site.pp文件位置
	fileserverconfig = /etc/puppet/fileserver.conf.kissdev  #设置环境的fileserver
	
	[kissmq]
	modulepath = $confdir/environments/kissmq/environment/modules:$confdir/environments/kisstest/application/modules
	manifest = $confdir/environments/kisstest/manifests/site.pp
	fileserverconfig = /etc/puppet/fileserver.conf.kisstest
	
	[kissprd]
	modulepath = $confdir/environments/kissprd/environment/modules:$confdir/environments/kissprd/application/modules
	manifest = $confdir/environments/kissprd/manifests/site.pp
	fileserverconfig = /etc/puppet/fileserver.conf.kissprd

**顺便解释一下：**为什么在每个环境下会有environment和application两个目录，其中environment目录是存放基础环境模块的，比如puppet、yum等；而application目录是存在应用环境模块的，比如apache、mysql等。当然也可以放在同一个目录下，如果应用多的话还可以将application进行拆分，一切都是为了方便管理而考虑。

## 2、创建多环境目录结构 ##

	[root@puppetmaster environments]# mkdir kissdev
	[root@puppetmaster environments]# mkdir kissdev/{application/modules,environment/modules} -p
	[root@puppetmaster environments]# tree .
	.
	└── kissdev
	    ├── application
	    │   └── modules   #存放应用的模块
	    └── environment
	        └── modules  #存放基础环境模块
	
	5 directories, 0 files
	[root@puppetmaster environments]# cp kissdev kissmq -rp
	[root@puppetmaster environments]# cp kissdev kissprd -rp
	[root@puppetmaster environments]# tree .
	.
	├── kissdev
	│   ├── application
	│   │   └── modules
	│   └── environment
	│       └── modules
	├── kissmq
	│   ├── application
	│   │   └── modules
	│   └── environment
	│       └── modules
	└── kissprd
	    ├── application
	    │   └── modules
	    └── environment
	        └── modules
	
	15 directories, 0 files

## 3、移动默认环境modules中的配置到kissprd对应的环境中 ##

其中puppet和yum模块属于基础环境模块，motd属于应用环境模块

	[root@puppetmaster environments]# mv /etc/puppet/modules/puppet kissprd/environment/modules/ 
	[root@puppetmaster environments]# mv /etc/puppet/modules/yum  kissprd/environment/modules/ 
	[root@puppetmaster environments]# mv /etc/puppet/modules/motd kissprd/application/modules/

## 4、复制manifests文件至kissprd环境中 ##

	[root@puppetmaster environments]# cp /etc/puppet/manifests kissprd/ -r

复制完成后整个环境如下

	[root@puppetmaster environments]# tree kissprd/
	kissprd/
	├── application
	│   └── modules
	│       └── motd
	│           ├── files
	│           │   └── etc
	│           │       └── motd
	│           ├── manifests
	│           │   └── init.pp
	│           └── templates
	├── environment
	│   └── modules
	│       ├── puppet
	│       │   ├── files
	│       │   ├── manifests
	│       │   │   ├── config.pp
	│       │   │   ├── init.pp
	│       │   │   ├── install.pp
	│       │   │   ├── params.pp
	│       │   │   └── service.pp
	│       │   └── templates
	│       │       └── puppet.conf.erb
	│       └── yum
	│           ├── files
	│           │   ├── etc
	│           │   │   └── yum.conf
	│           │   └── PM-GPG-KEY
	│           │       ├── RPM-GPG-KEY-puppet-release
	│           │       ├── RPM-GPG-KEY-redhat-release-rhel5
	│           │       └── RPM-GPG-KEY-redhat-release-rhel6
	│           ├── manifests
	│           │   ├── config.pp
	│           │   ├── init.pp
	│           │   ├── install.pp
	│           │   └── params.pp
	│           └── templates
	└── manifests
	    └── site.pp
	
	20 directories, 17 files

## 5、删除掉默认环境manifests中site.pp文件内容 ##

因为模块已经移除，其次默认环境production已经不再使用了。

	[root@puppetmaster environments]# >/etc/puppet/manifests/site.pp 

## 6、创建fileserverconfig文件 ##

	[root@puppetmaster ~]# cp /etc/puppet/fileserver.conf{,.kissdev}
	[root@puppetmaster ~]# cp /etc/puppet/fileserver.conf{,.kissqa}
	[root@puppetmaster ~]# cp /etc/puppet/fileserver.conf{,.kissprd}
	[root@puppetmaster ~]# ll /etc/puppet/
	total 88
	-rw-r--r-- 1 root root  2569 Jan  7 07:51 auth.conf
	-rw-r--r-- 1 root root    17 Mar  9 17:54 autosign.conf.bak
	drwxr-xr-x 5 root root  4096 Mar 27 22:33 environments
	-rw-r--r-- 1 root root   381 Jan  7 07:49 fileserver.conf
	-rw-r--r-- 1 root root   381 Mar 27 22:46 fileserver.conf.kissdev  #指向kissdev环境
	-rw-r--r-- 1 root root   381 Mar 27 22:46 fileserver.conf.kissprd  #指向kissmq环境
	-rw-r--r-- 1 root root   381 Mar 27 22:46 fileserver.conf.kissqa   #指向kissdev环境
	drwxr-xr-x 2 root root  4096 Mar 25 05:23 manifests
	drwxr-xr-x 2 root root  4096 Mar 27 22:40 modules
	-rw-r--r-- 1 root root  1063 Mar 27 21:55 puppet.conf
	-rw-r--r-- 1 root root   853 Mar  9 00:48 puppet.conf.bak
	-rw-r--r-- 1 root root 42031 Mar  9 03:25 puppet.conf.out

**7、重启puppetmaster服务**

	[root@puppetmaster ~]# /etc/init.d/puppetmaster restart
	Stopping puppetmaster:                                     [  OK  ]
	Starting puppetmaster:                                     [  OK  ]

**8、节点测试验证**

	[root@agent1 ~]# >/etc/motd 
	You have new mail in /var/spool/mail/root
	[root@agent1 ~]# puppet agent -t  #默认请求的是production环境，由于此环境里面没有模块所有不更新
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1395931884'
	notice: Finished catalog run in 0.02 seconds
	[root@agent1 ~]# puppet agent -t --environment=kissprd  #环境指向kissprd
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1395931962'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2014-03-27 22:52:27.000000000 +0800
	+++ /tmp/puppet-file20140327-26204-29bst1-0	2014-03-27 22:52:44.000000000 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket got a duplicate file {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	notice: Finished catalog run in 0.68 seconds
	[root@agent1 ~]# cat /etc/motd 
	--                       --
	--------puppet test---------
	--                       --

**9、节点更改环境**

如果节点是主动同步的方式，应该在puppet.conf文件中添加environment配置

	[root@agent1 ~]# vim /etc/puppet/puppet.conf
	
	### config by  puppet ###
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com
	    certname = agent1_cert.kisspuppet.com
	    runinterval = 10
	    environment =kissprd   #添加默认环境为kissprd

**10、继续测试**

	[root@agent1 ~]# puppet agent -t
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1395931962'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2014-03-27 22:55:43.000000000 +0800
	+++ /tmp/puppet-file20140327-30010-8ada2g-0	2014-03-27 22:56:19.000000000 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket got a duplicate file {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: 
	--- /etc/puppet/puppet.conf	2014-03-27 22:56:14.000000000 +0800
	+++ /tmp/puppet-file20140327-30010-cmjg48-0	2014-03-27 22:56:19.000000000 +0800
	@@ -10,4 +10,3 @@
	     server = puppetmaster.kisspuppet.com
	     certname = agent1_cert.kisspuppet.com
	     runinterval = 10
	-    environment =kissprd
	
	info: FileBucket got a duplicate file {md5}43df60b1aa2638c5f10aa7e6be892b77
	info: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]: Filebucketed /etc/puppet/puppet.conf to puppet with sum 43df60b1aa2638c5f10aa7e6be892b77
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: content changed '{md5}43df60b1aa2638c5f10aa7e6be892b77' to '{md5}8c67cb8c039bb6436556b91f0c6678c4'
	info: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]: Scheduling refresh of Class[Puppet::Service]
	info: Class[Puppet::Service]: Scheduling refresh of Service[puppet]
	notice: /Service[puppet]/ensure: ensure changed 'stopped' to 'running'
	notice: /Service[puppet]: Triggered 'refresh' from 1 events
	notice: Finished catalog run in 0.68 seconds
	[root@agent1 ~]# cat /etc/motd 
	--                       --
	--------puppet test---------
	--                       --

**备注：** 记得设置puppet模块中的puppet.conf.erb模板，否则会被还原哦。

## 后续问题 ##

1、puppetmaster端有三套环境，那么如何管理呢，接下来就应该考虑版本控制系统了，这里已经有写了[http://rsyslog.org/2013/11/16/svn-puppet/](http://rsyslog.org/2013/11/16/svn-puppet/)

2、后面讲的hiear中关于设置的变量对应到每个环境中是如何解决的。

关于多环境的部署有不理解的还可以参考书籍《精通Puppet配置管理工具》或者官网



