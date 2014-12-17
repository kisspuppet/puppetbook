## foreman架构的引入7-Foreman结合mcollective完成push动作 ##

**注：**以下内容是在**foreman1.6.3+puppet2.6.2**环境下进行操作。更多配置请参考官网[http://theforeman.org/manuals/1.6/index.html](http://theforeman.org/manuals/1.6/index.html)

在foreman-proxy的1.6.3版本，至少提供了以下五种触发puppet agent命令的工具，默认使用的是puppetrun，不过已经过时，这里介绍如何使用mcollective进行触发，下个章节会介绍如何使用puppetssh触发。

	#   puppetrun   (for puppetrun/kick, deprecated in Puppet 3)
	#   mcollective (uses mco puppet)
	#   puppetssh   (run puppet over ssh)
	#   salt        (uses salt puppet.run)
	#   customrun   (calls a custom command with args)

<!--more-->

在整个测试之前，首先要保障你的mco+mq在命令行操作的情况下是OK的。如果没有OK或者不懂什么是mco+mq，请参考之前的文章。

如何是OK的？如下：

	[root@puppetmaster162 yum.repos.d]# mco puppet -v runonce
	Discovering hosts using the mc method for 2 second(s) .... 1
	
	 * [ ============================================================> ] 1 / 1
	
	
	puppetmaster162.kisspuppet.com          : OK
	    {:summary=>      "Started a Puppet run using the 'puppet agent --test --color=false --splay --splaylimit 30' command"}
	
	
	
	---- rpc stats ----
	           Nodes: 1 / 1
	     Pass / Fail: 1 / 0
	      Start Time: Wed Dec 17 16:22:15 +0800 2014
	  Discovery Time: 2004.22ms
	      Agent Time: 71.49ms
	      Total Time: 2075.70ms

## 1、在Foreman中开启puppet插件的puppetrun功能 ##

![Foreman安装](http://kisspuppet.com/img/foreman07-1.png)

![Foreman安装](http://kisspuppet.com/img/foreman07-2.png)


## 2、配置foreman-proxy代理的puppet的puppet_provider##

	[root@puppetmaster162 ~]# vim /etc/foreman-proxy/settings.d/puppet.yml 
	---
	# Puppet management
	:enabled: true
	:puppet_conf: /etc/puppet/puppet.conf
	# valid providers:
	#   puppetrun   (for puppetrun/kick, deprecated in Puppet 3)
	#   mcollective (uses mco puppet)
	#   puppetssh   (run puppet over ssh)
	#   salt        (uses salt puppet.run)
	#   customrun   (calls a custom command with args)
	:puppet_provider: mcollective
	...

## 3、配置sudoer，添加mco命令 ##

	[root@puppetmaster162 ~]# vim /etc/sudoers.d/foreman-proxy 
	
	foreman-proxy ALL = NOPASSWD : /usr/bin/puppet cert *, /usr/bin/mco puppet runonce *
	Defaults:foreman-proxy !requiretty

	[root@puppetmaster162 ~]# /etc/init.d/foreman-proxy restart
	Stopping foreman-proxy:                                    [  OK  ]
	Starting foreman-proxy:                                    [  OK  ]


## 4、页面测试puppetrun按钮 ##

![Foreman安装](http://kisspuppet.com/img/foreman07-3.png)


成功之后的显示

![Foreman安装](http://kisspuppet.com/img/foreman07-4.png)


## 5、查看报告看更详细的信息 ##


	#可以通过日志查看执行情况
	[root@puppetmaster162 yum.repos.d]# tailf  /var/log/foreman-proxy/proxy.log
	
	
	192.168.20.11 - - [17/Dec/2014 16:25:36] "POST /run HTTP/1.1" 200 - 0.5454

	以上

	[root@puppetmaster162 ~]# cat /etc/foreman-proxy/settings.yml
	...
	:log_file: /var/log/foreman-proxy/proxy.log
	# valid options are
	# WARN, DEBUG, Error, Fatal, INFO, UNKNOWN
	:log_level: DEBUG  #开启debug模式，显示更详细的信息，排错的时候使用。1.5版本之前默认是开启的


	[root@puppetmaster162 yum.repos.d]# tailf  /var/log/foreman-proxy/proxy.log
	I, [2014-12-17T16:27:43.148519 #24337]  INFO -- : 'foreman_proxy' settings were initialized with default values: :enabled: true
	W, [2014-12-17T16:27:43.155592 #24337]  WARN -- : Couldn't find settings file /etc/foreman-proxy/settings.d/facts.yml. Using default settings.
	I, [2014-12-17T16:27:43.155860 #24337]  INFO -- : 'facts' settings were initialized with default values: :enabled: true
	I, [2014-12-17T16:27:43.163012 #24337]  INFO -- : 'dns' module is disabled.
	I, [2014-12-17T16:27:43.163513 #24337]  INFO -- : 'tftp' module is disabled.
	I, [2014-12-17T16:27:43.163933 #24337]  INFO -- : 'dhcp' module is disabled.
	I, [2014-12-17T16:27:43.579571 #24337]  INFO -- : 'puppet' settings were initialized with default values: :puppetdir: /etc/puppet
	I, [2014-12-17T16:27:43.583486 #24337]  INFO -- : 'bmc' module is disabled.
	I, [2014-12-17T16:27:43.583655 #24337]  INFO -- : 'chefproxy' module is disabled.
	I, [2014-12-17T16:27:43.583934 #24337]  INFO -- : 'realm' module is disabled.
	D, [2014-12-17T16:28:15.059328 #24344] DEBUG -- : about to execute: /usr/bin/sudo -u root /usr/bin/mco puppet runonce -I puppetmaster162.kisspuppet.com
	192.168.20.11 - - [17/Dec/2014 16:28:15] "POST /run HTTP/1.1" 200 - 0.5468


失败的情况如下：

![Foreman安装](http://kisspuppet.com/img/foreman07-5.png)

	[root@puppetmaster162 ~]# tailf  /var/log/foreman-proxy/proxy.log
	I, [2014-12-17T16:27:43.163933 #24337]  INFO -- : 'dhcp' module is disabled.
	I, [2014-12-17T16:27:43.579571 #24337]  INFO -- : 'puppet' settings were initialized with default values: :puppetdir: /etc/puppet
	I, [2014-12-17T16:27:43.583486 #24337]  INFO -- : 'bmc' module is disabled.
	I, [2014-12-17T16:27:43.583655 #24337]  INFO -- : 'chefproxy' module is disabled.
	I, [2014-12-17T16:27:43.583934 #24337]  INFO -- : 'realm' module is disabled.
	D, [2014-12-17T16:28:15.059328 #24344] DEBUG -- : about to execute: /usr/bin/sudo -u root /usr/bin/mco puppet runonce -I puppetmaster162.kisspuppet.com
	192.168.20.11 - - [17/Dec/2014 16:28:15] "POST /run HTTP/1.1" 200 - 0.5468
	D, [2014-12-17T16:32:56.924849 #24344] DEBUG -- : about to execute: /usr/bin/sudo -u root /usr/bin/mco puppet runonce -I puppetmaster162.kisspuppet.com
	192.168.20.11 - - [17/Dec/2014 16:32:57] "POST /run HTTP/1.1" 200 - 0.6095
	D, [2014-12-17T16:32:57.878231 #24344] DEBUG -- : about to execute: /usr/bin/sudo -u root /usr/bin/mco puppet runonce -I foreman163.kisspuppet.com
	W, [2014-12-17T16:33:20.364704 #24344]  WARN -- : Non-null exit code when executing '/usr/bin/sudo-uroot/usr/bin/mcopuppetrunonce-Iforeman163.kisspuppet.com'
	E, [2014-12-17T16:33:20.368673 #24344] ERROR -- : Failed puppet run: Check Log files
	192.168.20.11 - - [17/Dec/2014 16:33:20] "POST /run HTTP/1.1" 500 34 22.4920


**备注：**Foreman在命令执行后的显示这块做的其实很不好的，如何能够将所有节点执行的情况动态或者显示在界面上就更好了！