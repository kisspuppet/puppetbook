#### Puppet扩展篇3-如何扩展master的SSL传输性能（apache）



**描述：**puppet使用SSL(https)协议来进行通讯，默认情况下，puppet server端使用基于Ruby的WEBRick HTTP服务器。由于WEBRick HTTP服务器在处理agent端的性能方面并不是很强劲，因此需要扩展puppet，搭建Apache或者其他强劲的web服务器来处理客户的https请求。
<!--more-->

**需要解决的问题：**

- 扩展传输方式：提高性能并增加Master和agent之间的并发连接数量。
- 扩展SSL：采用良好的SSL证书管理方法来加密Master和agent之间的通讯。
<!--more-->

**参考：**[http://projects.puppetlabs.com/projects/1/wiki/Using_Passenger](http://projects.puppetlabs.com/projects/1/wiki/Using_Passenger)

## 1 使用Ruby Gem安装Passenger ##

	[root@puppetserver etc]# yum install ruby-devel ruby-libs rubygems libcurl-devel
	[root@puppetserver etc]# yum install httpd httpd-devel apr-util-devel apr-devel mod_ssl
	[root@puppetserver repos]# gem install --local passenger-4.0.19.gem #自动解决依赖关系，进入gem包目录进行安装
	Building native extensions.  This could take a while...
	Successfully installed rake-10.0.1
	Successfully installed daemon_controller-1.1.5
	Successfully installed rack-1.5.2
	Successfully installed passenger-4.0.19

## 2 整合Apache和Passenger ##

	[root@puppetserver rpms]# yum install gcc-c++  gcc openssl-devel #源码包编译安装（安装需要apache gcc gcc-c++ openssl-devel开发包的支持）
	[root@puppetserver etc]# passenger-install-apache2-module #按照相关提示解决依赖关系，安装完成之后会显示
	…
	The Apache 2 module was successfully installed.
	Please edit your Apache configuration file, and add these lines:
	   LoadModule passenger_module /usr/lib/ruby/gems/1.8/gems/passenger-4.0.19/buildout/apache2/mod_passenger.so
	   PassengerRoot /usr/lib/ruby/gems/1.8/gems/passenger-4.0.19
	   PassengerDefaultRuby /usr/bin/ruby
	
	After you restart Apache, you are ready to deploy any number of Ruby on Rails
	applications on Apache, without any further Ruby on Rails-specific
	configuration!
	…

## 3 配置Apache和Passenger ##

创建虚拟主机并加载passenger相关模块，注意证书路径要和puppet实际证书路径对应。虚拟主机配置Apache以监听在8140端口，并且使用SSL和Puppet Master生成的证书对所有通讯进行加密。同时还将配置Passenger来使系统的Ruby解释器并且提供Rack配置文件`config.ru`的路径

	[root@puppetserver conf.d]# vim passenger.conf
	LoadModule passenger_module /usr/lib/ruby/gems/1.8/gems/passenger-4.0.19/buildout/apache2/mod_passenger.so
	<IfModule mod_passenger.c>
	   PassengerRoot /usr/lib/ruby/gems/1.8/gems/passenger-4.0.19
	   PassengerRuby /usr/bin/ruby
	   PassengerHighPerformance on
	   PassengerMaxPoolSize 12
	   PassengerPoolIdleTime 1500
	   PassengerStatThrottleRate 120
	 # RailsAutoDetect On
	</IfModule>
	Listen 8140  #监听TCP 8140端口，这是PuppetMaster服务器的标准端口
	<VirtualHost *:8140>
	        SSLEngine on  #开始ssl加密
	        SSLProtocol -ALL +SSLv3 +TLSv1
	        SSLCipherSuite ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:-LOW:-SSLv2:-EXP #开启ssl加密
	
	        SSLCertificateFile      /var/lib/puppet/ssl/certs/puppetserver.kisspuppet.com.pem 
	        SSLCertificateKeyFile   /var/lib/puppet/ssl/private_keys/puppetserver.kisspuppet.com.pem
	        SSLCertificateChainFile /var/lib/puppet/ssl/ca/ca_crt.pem
	        SSLCACertificateFile    /var/lib/puppet/ssl/ca/ca_crt.pem
	        SSLCARevocationFile     /var/lib/puppet/ssl/ca/ca_crt.pem #打开证书撤销功能，当我们颁发或撤销Puppet agent的证书时，Puppet cert命令会自动更关心ca_crl.pem文件
	        SSLVerifyClient optional
	        SSLVerifyDepth  1
	        SSLOptions +StdEnvVars #配置Apache来验证Puppet agent证书的真实性。验证的结果会被保存在这个环境变量中，运行在Passenger中的Puppet master进程会使用这个变量来认证Puppet agent。
	
	#Puppet agent证书验证的结果会以客户端请求头的形式存放在标准环境中。
	        RequestHeader unset X-Forwarded-For
	        RequestHeader set X-SSL-Subject %{SSL_CLIENT_S_DN}e
	        RequestHeader set X-Client-DN %{SSL_CLIENT_S_DN}e
	        RequestHeader set X-Client-Verify %{SSL_CLIENT_VERIFY}e
	
	        DocumentRoot /etc/puppet/rack/puppetmaster/public/
	        RackBaseURI /
	#Rack为Web服务器提供了用来和Puppet这样的Ruby HTTP服务交换请求和响应的一些常用API。Rack经常被用于在多台Web服务器上部署如Puppet Dashboad这样的web程序。
	        <Directory /etc/puppet/rack/puppetmaster/>  #虚拟主机部分
	                Options None
	                AllowOverride None
	                Order allow,deny
	                allow from all
	        </Directory>
	</VirtualHost>

---

	[root@c1.inanu.net]# service httpd configtest  #检查apache配置语法是否正确
	Warning: DocumentRoot [/etc/puppet/rack/puppetmaster/public/] does not exist
	Syntax OK
**备注：**有关puppet虚拟主机配置可参考默认配置

	/usr/share/puppet/ext/rack/files/apache2.conf

## 4 准备config.ru配置文件 ##

	[root@puppetserver rack]# mkdir -p /etc/puppet/rack/puppetmaster/{public,tmp} #为Rack和Puppet master的rack程序实例创建框架目录。
	[root@puppetserver rack]# cp /usr/share/puppet/ext/rack/files/config.ru /etc/puppet/rack/puppetmaster/ 
	[root@puppetserver rack]# vim /etc/puppet/rack/puppetmaster/config.ru  #默认即可
	
	# a config.ru, for use with every rack-compatible webserver.
	# SSL needs to be handled outside this, though.
	
	# if puppet is not in your RUBYLIB:
	# $:.unshift('/opt/puppet/lib')
	
	$0 = "master"
	
	# if you want debugging:
	# ARGV << "--debug"
	
	ARGV << "--rack"
	require 'puppet/application/master'
	# we're usually running inside a Rack::Builder.new {} block,
	# therefore we need to call run *here*.
	run Puppet::Application[:master].run 

**备注：**
如果需要最新的Rack配置文件，可以在Puppet最新发行版的ext目录找到。也可以在[https://github.com/puppetlabs/puppet/tree/master/ext/rack/files](https://github.com/puppetlabs/puppet/tree/master/ext/rack/files)找到。

	[root@puppetserver rack]# chown puppet. /etc/puppet/rack/puppetmaster/config.ru #Rack配置文件config.ru的用户和组应该是puppet。当Apache启动时，Passenger会检查这个文件的所有者，并将其使用的账号从root切换到权限较低的puppet账户。

## 5 在Apache中测试PuppetMaster ##

	[root@puppetserver ~]# /etc/rc.d/init.d/puppetmaster stop #停止puppetmaster进程
	[root@puppetserver ~]# chkconfig puppetmaster off #防止开机自动启动
	[root@puppetserver ~]# /etc/rc.d/init.d/httpd start #启动apache服务
	[root@puppetserver ~]# chkconfig httpd off  #设置开机自动启动
	[root@puppetserver ~]# netstat -nlp | grep 8140  #监听8140端口
	tcp        0      0 :::8140             :::*          LISTEN      4162/httpd

**测试一：**通过浏览器（IE版本<9）访问https://172.16.200.100:8140/，出现以下信息，说明配置正确

![apache+passenger替代WEBrick](http://kisspuppet.com/img/apache-passenger-1.png)

**测试二：**在节点上运行puppet程序，在服务器端通过apache访问日志查看是否有puppet的请求，如果返回状态吗`“200”`表明这次请求时成功的。

	[root@puppetserver conf.d]# tailf  /var/log/httpd/access_log
	172.16.200.101 - - [22/Jul/2013:10:30:34 +0800] "GET /production/file_metadata/modules/mysql/etc/my.cnf? HTTP/1.1" 200 298 "-" "-"
	172.16.200.101 - - [22/Jul/2013:10:30:34 +0800] "GET /production/file_metadata/modules/motd/etc/motd? HTTP/1.1" 200 295 "-" "-"
	172.16.200.101 - - [22/Jul/2013:10:30:35 +0800] "PUT /production/report/agent1.kisspuppet.com HTTP/1.1" 200 14 "-" "-"
	172.16.200.101 - - [22/Jul/2013:10:30:40 +0800] "POST /production/catalog/agent1.kisspuppet.com HTTP/1.1" 200 8346 "-" "-"
	172.16.200.101 - - [22/Jul/2013:10:30:41 +0800] "GET /production/file_metadata/modules/ssh/etc/ssh/sshd_config? HTTP/1.1"
	
