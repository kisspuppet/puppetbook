#### MCollective架构篇2-MCollective+MQ架构的部署


# 1 Mcollective介绍 #

MCollective 是一个构建服务器编排(Server Orchestration)和并行工作执行系统的框架。 首先，MCollective 是一种针对服务器集群进行可编程控制的系统管理解决方案。在这一点上，它的功能类似：Func，Fabric 和 Capistrano。

其次，MCollective 的设计打破基于中心存储式系统和像 SSH 这样的工具，不再仅仅痴迷于 SSH 的 For 循环。它使用发布订阅中间件(Publish Subscribe Middleware)这样的现代化 工具和通过目标数据(meta data)而不是主机名(hostnames)来实时发现网络资源这样的现代化理念。提供了一个可扩展的而且迅速的并行执行环境。

MCollective 工具为命令行界面，但它可与数千个应用实例进行通信，而且传输速度惊人。无论部署的实例位于什么位置，通信都能以线速进行传输，使用的是一个类似多路传送的推送信息系统。MCollective 工具没有可视化用户界面，用户只能通过检索来获取需要应用的实例。Puppet Dashboard 提供有这部分功能。

<!--more-->

# 2 安装和配置RabbitMQ #

## 2.1 安装和配置RabbitMQ ##

**2.1.1 安装RabbitMQ**

	[root@puppetserver rpms]# yum install erlang #RabbitMQ依赖erlang语言，需要安装大概65个左右的erlang依赖包
	[root@puppetserver rpms]# yum install rabbitmq-server
	[root@puppetserver rpms]# ll /usr/lib/rabbitmq/lib/rabbitmq_server-3.1.5/plugins/ #默认已经安装了stomp插件，老版本需要下载安装
	-rw-r--r-- 1 root root 242999 Aug 24 17:42 amqp_client-3.1.5.ez
	-rw-r--r-- 1 root root  85847 Aug 24 17:42 rabbitmq_stomp-3.1.5.ez
	…

**2.1.2 启动rabbitmq-server**

	[root@puppetserver rpms]# /etc/rc.d/init.d/rabbitmq-server start #启动rabbitmq服务
	Starting rabbitmq-server: SUCCESS
	rabbitmq-server.
	[root@puppetserver rpms]# /etc/rc.d/init.d/rabbitmq-server status #查看rabbitmq状态
	Status of node rabbit@puppetserver ...
	[{pid,43198},
	 {running_applications,[{rabbit,"RabbitMQ","3.1.5"},
	                        {mnesia,"MNESIA  CXC 138 12","4.5"},
	                        {os_mon,"CPO  CXC 138 46","2.2.7"},
	                        {xmerl,"XML parser","1.2.10"},
	                        {sasl,"SASL  CXC 138 11","2.1.10"},
	                        {stdlib,"ERTS  CXC 138 10","1.17.5"},
	                        {kernel,"ERTS  CXC 138 10","2.14.5"}]},
	 {os,{unix,linux}},
	 {erlang_version,"Erlang R14B04 (erts-5.8.5) [source] [64-bit] [rq:1] [async-threads:30] [kernel-poll:true]\n"},
	 {memory,[{total,27101856},
	          {connection_procs,2648},
	          {queue_procs,5296},
	          {plugins,0},
	          {other_proc,9182320},
	          {mnesia,57456},
	          {mgmt_db,0},
	          {msg_index,21848},
	          {other_ets,765504},
	          {binary,3296},
	          {code,14419185},
	          {atom,1354457},
	          {other_system,1289846}]},
	 {vm_memory_high_watermark,0.4},
	 {vm_memory_limit,838362726},
	 {disk_free_limit,1000000000},
	 {disk_free,15992676352},
	 {file_descriptors,[{total_limit,924},
	                    {total_used,3},
	                    {sockets_limit,829},
	                    {sockets_used,1}]},
	 {processes,[{limit,1048576},{used,122}]},
	 {run_queue,0},
	 {uptime,4}]
	...done.
	[root@puppetserver rpms]# netstat -nlp | grep beam #默认监听端口为5672
	tcp        0      0 0.0.0.0:44422               0.0.0.0:*                   LISTEN      43198/beam          
	tcp        0      0 :::5672                     :::*                        LISTEN      43198/beam   

**2.1.3 配置RabbitMQ**

**2.1.3.1 加载amqp_client和rabbit_stomp插件**

	[root@puppetserver sbin]#ln -s /usr/lib/rabbitmq/lib/rabbitmq_server-3.1.5/sbin/rabbitmq-plugins /usr/sbin/rabbitmq-plugins #创建命令rabbitmq-plugins的软连接
	[root@puppetserver sbin]# ln -s /usr/lib/rabbitmq/lib/rabbitmq_server-3.1.5/sbin/rabbitmq-env /usr/sbin/rabbitmq-env #创建命令rabbitmq-env的软连接
	[root@puppetserver sbin]# rabbitmq-plugins enable rabbitmq_stomp #开启rabbitmq_stomp插件
	The following plugins have been enabled:
	  amqp_client
	  rabbitmq_stomp
	Plugin configuration has changed. Restart RabbitMQ for changes to take effect.
	[root@puppetserver sbin]# /etc/rc.d/init.d/rabbitmq-server restart
	Restarting rabbitmq-server: SUCCESS
	rabbitmq-server.
	[root@puppetserver rabbitmq]# tailf /var/log/rabbitmq/rabbit\@puppetserver.log #可以从日志看到stomp插件加载成功
	=INFO REPORT==== 3-Oct-2013::20:25:18 ===
	started STOMP TCP Listener on [::]:61613
	=INFO REPORT==== 3-Oct-2013::20:25:18 ===
	Server startup complete; 2 plugins started.
	 * amqp_client
	 * rabbitmq_stomp
 * 

**2.1.3.2 创建rabbitmq.config配置文件，修改监听端口为61613**

	[root@puppetserver rpms]# vim /etc/rabbitmq/rabbitmq.config
	[
	        {stomp,[ {tcp_listeners, [61613]} ]} #设置connector为stomp，监听端口为61613
	]. 
	[root@puppetserver rpms]# /etc/rc.d/init.d/rabbitmq-server restart
	Restarting rabbitmq-server: SUCCESS
	rabbitmq-server.
	[root@puppetserver rpms]# netstat -nlp | grep beam  #默认监听端口为61613
	tcp        0      0 0.0.0.0:56532               0.0.0.0:*                   LISTEN      1906/beam.smp       
	tcp        0      0 :::61613                    :::*                        LISTEN      1906/beam.smp       
	tcp        0      0 :::5672                     :::*                        LISTEN      1906/beam.smp   

**2.1.3.3 删除默认账户guest，为MCollective创建账户“mcollective”并设置密码为“secret”，然后设置权限。**

	[root@puppetserver rpms]# rabbitmqctl delete_user guest
	Deleting user "guest" ...
	...done.
	[root@puppetserver rpms]# rabbitmqctl add_user mcollective secret
	Creating user "mcollective" ...
	...done.
	[root@puppetserver rpms]# rabbitmqctl set_permissions -p "/" mcollective  ".*" ".*" ".*"
	Setting permissions for user "mcollective" in vhost "/" ...
	...done.
	[root@puppetserver sbin]# rabbitmqctl list_users #查看监听用户
	Listing users ...
	mcollective    []
	...done.

**备注：**RabbitMQ拥有一个默认的guest账户，它默认对消息队列拥有全部权限。出于安全方面的考虑，建议删除这个账户。

更多详细配置信息请参考 http://www.rabbitmq.com/admin-guide.html

更多详细配置信息请参考： http://docs.puppetlabs.com/mcollective/reference/plugins/connector_rabbitmq.html

# 3 安装和配置MCollective #
## 3.1 安装MCollective ##

**3.1.1 测试端安装MCollective客户端**

	[root@puppetserver rpms]# yum install mcollective-common  mcollective-client #依赖包rubygem-stomp    

**3.1.2 节点安装MCollective服务端**

	[root@agent1 ~]# yum install mcollective  mcollective-common #依赖rubygem-stomp、rubygems和ruby相关包

## 3.2 配置MCollective ##

**3.2.1 测试端配置MCollective客户端**

	[root@puppetserver rpms]# vim /etc/mcollective/client.cfg
	topicprefix = /topic/
	main_collective = mcollective
	collectives = mcollective
	libdir = /usr/libexec/mcollective
	logger_type = console
	loglevel = warn
	# Plugins
	securityprovider = psk
	plugin.psk = a36cd839414370e10fd281b8a38a4f48 #MCollective通信共享密钥，和MCollective服务端保持一致
	connector = stomp  #通信协议
	plugin.stomp.host = 192.168.100.110  #Middleware地址
	plugin.stomp.port = 61613  #Middleware监听端口
	plugin.stomp.user = mcollective  #Middleware通信账号
	plugin.stomp.password = secret #Middleware通信密码
	# Facts
	factsource = yaml
	plugin.yaml = /etc/mcollective/facts.yaml

**3.2.2 节点配置MCollective服务端**

	[root@agent1 rpms]# vim /etc/mcollective/server.cfg 
	topicprefix = /topic/
	main_collective = mcollective
	collectives = mcollective
	libdir = /usr/libexec/mcollective  #存放plugins的位置
	logfile = /var/log/mcollective.log
	loglevel = info
	daemonize = 1
	# Plugins
	securityprovider = psk
	plugin.psk = a36cd839414370e10fd281b8a38a4f48 #MCollective通信共享密钥，和MCollective客户端保持一致
	connector = stomp #通信协议
	plugin.stomp.host =  192.168.100.110  #Middleware地址
	plugin.stomp.port =  61613  #Middleware监听端口
	plugin.stomp.user = mcollective  #Middleware通信账号
	plugin.stomp.password = secret #Middleware通信密码
	# Facts
	factsource = yaml
	plugin.yaml = /etc/mcollective/facts.yaml
	[root@agent1 ~]# /etc/rc.d/init.d/mcollective start
	Starting mcollective:                                      [  OK  ]
	[root@agent1 ~]# chkconfig mcollective on
	[root@agent1 ~]#

## 3.3 测试Mcollective与Middleware通信 ##

	[root@puppetserver rpms]# mco ping  #检查所有存活的节点
	agent2.kisspuppet.com                       time=119.98 ms
	agent1.kisspuppet.com                       time=159.31 ms
	---- ping statistics ----
	2 replies max: 159.31 min: 119.98 avg: 139.64 
	[root@puppetserver rpms]# mco find
	agent1.kisspuppet.com
	agent2.kisspuppet.com
