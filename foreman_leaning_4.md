#### foreman架构的引入4-安装Foreman1.6.3架构（foreman与puppetmaster分离）


**注意：**本实验是在离线情况下安装的，所以需要在本地创建自己的yum仓库，创建方法可参考《[如何根据版本制作属于自己的puppet yum源](http://kisspuppet.com/2014/01/26/puppet_create_repo/)》，如何你实在是比较懒或者搞不定rpm包之间的依赖关系，那就去我的github上下载吧：[https://github.com/kisspuppet/foreman-repo](https://github.com/kisspuppet/foreman-repo)

更多安装细节请参考官网：[http://theforeman.org/manuals/1.6/index.html](http://theforeman.org/manuals/1.6/index.html)

之前讲的all-in-one方式建议只用于测试使用，如果要用于生产环境，建议将foreman和puppetmaster分离安装，更有利于后期的维护和扩展。还有就是之前你已经部署过puppetmaster了，如何单独部署foreman和puppetmaster通信也是值得考虑的问题。



## 1、软件包的选型如下： ##
- **puppet-server     3.6.2**
- **puppet            3.6.2**
- **facter            2.0.2**
- **mcollective       2.2.4**
- **rabbitmq-server   3.2.4**
- **foreman           1.6.3**
- **foreman-proxy     1.6.3**

## 2、系统环境准备 ##


| 角色         |        主机名                 |   系统版本     |        IP        |
| -------------|:-----------------------------:| --------------:|-----------------:|
| foreman      | foreman163.kisspuppet.com     | rhel6.4-x86_64 | 192.168.20.11/24 |
| puppetmaster | pupptmaster162.kisspuppet.com | rhel6.4-x86_64 | 192.168.20.12/24 |


## 3、安装puppetmaster ##

**3.1、安装puppetmaster，并生成CA和证书**

	[root@puppetmaster162 ~]# yum install puppet puppet-server facter
	
	[root@puppetmaster162 puppet]# vim /etc/puppet/puppet.conf
    [agent]
    server = puppetmaster162.kisspuppet.com
    pluginsync = false
    ...
	[master]
    certname = puppetmaster162.kisspuppet.com
    environmentpath  = /etc/puppet/environments
    basemodulepath   = /etc/puppet/modules:/usr/share/puppet/modules  
	environment_timeout = 10
    
	[root@puppetmaster162 ~]# /etc/init.d/puppetmaster start
	Starting puppetmaster:                                     [  OK  ]
	[root@puppetmaster162 ~]# puppet cert --list --all
	+ "puppetmaster162.kisspuppet.com" (SHA256) 2E:B3:73:4F:CD:EE:0C:64:2C:DF:24:E6:D3:62:F3:1C:AC:A3:28:60:67:1D:0C:8C:C5:CA:68:5B:4B:2F:49:B9 (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster162.kisspuppet.com")

**3.2、测试puppetmaster是否能够正常使用**

	[root@puppetmaster162 ~]# puppet agent -t
	Info: Caching catalog for puppetmaster162.kisspuppet.com
	Info: Applying configuration version '1417749612'
	Notice: Finished catalog run in 0.04 seconds

**注：**以上安装方式，puppetmaster工作在Webrick上，性能非常差，需要更换为性能好的web服务器上，如果更换，请参考 [http://kisspuppet.com/2014/10/18/puppet_learning_ext3/](http://kisspuppet.com/2014/10/18/puppet_learning_ext3/)  [http://kisspuppet.com/2014/10/20/puppet_learning_ext4/](http://kisspuppet.com/2014/10/20/puppet_learning_ext4/)


## 4、安装Foreman ##

**4.1、安装puppet客户端并完成认证**
     
    #安装
	[root@foreman163 ~]# yum install puppet facter
	[root@foreman163 ~]# vim /etc/puppet/puppet.conf 
    [main]
    ...
    privatekeydir = $ssldir/private_keys { group = service }  
    hostprivkey = $privatekeydir/$certname.pem { mode = 640 }

    [agent]
	server = puppetmaster162.kisspuppet.com
	pluginsync = false
	
    #申请认证
	[root@foreman163 ~]# puppet  agent -t
	Info: Creating a new SSL key for foreman163.kisspuppet.com
	Info: Caching certificate for ca
	Info: csr_attributes file loading from /etc/puppet/csr_attributes.yaml
	Info: Creating a new SSL certificate request for 	
	Info: Certificate Request fingerprint (SHA256): 35:5D:E5:74:71:E0:FD:D2:67:34:17:92:3D:60:F2:A1:34:26:BA:E5:2D:1F:3A:0E:07:6F:85:38:A8:39:8B:65
	Info: Caching certificate for ca
	Exiting; no certificate found and waitforcert is disabled
	
    #授权证书
	[root@puppetmaster162 ~]# puppet cert --sign foreman163.kisspuppet.com
	Notice: Signed certificate request for foreman163.kisspuppet.com
	Notice: Removing file Puppet::SSL::CertificateRequest foreman163.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/foreman163.kisspuppet.com.pem'
	
    #测试
	[root@foreman163 ~]# puppet  agent -t
	Info: Caching catalog for foreman163.kisspuppet.com
	Info: Applying configuration version '1417749612'
	Notice: Finished catalog run in 0.05 seconds

**4.2、通过foreman-installer安装foreman**

foreman默认安装选择的数据库为postgresql，这里选用mysql进行安装。

**注意：**openssl版本要升级到1.0.1e版本
	
    #先安装包
	[root@foreman163 ~]# yum install foreman-installer foreman  mod_passenger mod_ssl ruby193-rubygem-passenger-native mysql mysql-server foreman-mysql2   openssl
    ...
	Updated:
	  openssl.x86_64 0:1.0.1e-15.el6                                                   
	
	Replaced:
	  ruby193-v8.x86_64 1:3.14.5.10-2.el6                                              
	
	Complete!
    
    #然后通过foreman-installer调用puppet进行配置
	[root@foreman163 ~]# foreman-installer --foreman-db-adapter mysql2 --foreman-db-type mysql --no-enable-puppet --no-enable-foreman-proxy --foreman-configure-epel-repo=false
	Installing             Done                                               [100%] []
	  Success!
	  * Foreman is running at https://foreman163.kisspuppet.com
	      Initial credentials are admin / 2kWcqJsW6cLDwo7m
	  The full log is at /var/log/foreman-installer/foreman-installer.log

**注：**以上安装完成之后，默认登录密码为随机密码，这跟之前版本有所不同。

安装完成之后，通过火狐或者谷歌浏览器访问看是否安装成功https://192.168.20.11
 
![Foreman安装](http://kisspuppet.com/img/foreman04-2.jpg)

![Foreman安装](http://kisspuppet.com/img/foreman04-3.jpg)

记得修改默认密码，否则待会忘了又登录不了了。
![Foreman安装](http://kisspuppet.com/img/foreman04-4.jpg)


## 5、安装Foreman-proxy ##

**注：**这里的foreman-proxy主要是代理puppet以及puppetca，所以要安装在puppetmaster上。

**5.1、安装foreman-proxy**

	[root@puppetmaster162 ~]# yum install foreman-installer foreman-proxy tftp-server syslinux

	[root@puppetmaster162 yum.repos.d]# foreman-installer --no-enable-foreman --no-enable-foreman-cli --no-enable-foreman-plugin-bootdisk --no-enable-foreman-plugin-setup --no-enable-puppet --enable-foreman-proxy  --foreman-proxy-puppetrun=true  --foreman-proxy-puppetrun-provider=mcollective  --foreman-proxy-puppetca=true   --foreman-proxy-dhcp=false  --foreman-proxy-tftp=false  --foreman-proxy-dns=false --foreman-proxy-register-in-foreman=false  --foreman-configure-epel-repo=false --foreman-configure-scl-repo=false
	Installing             Done                                               [100%] []
	  Success!
	  * Foreman Proxy is running at https://puppetmaster162.kisspuppet.com:8443
	  The full log is at /var/log/foreman-installer/foreman-installer.log

	#检测8443端口
	[root@puppetmaster162 ~]# netstat -nlatp | grep 8443
	tcp        0      0 0.0.0.0:8443                0.0.0.0:*                   LISTEN      4635/ruby           

**5.2、设置ENC**

    #从foreman-installer中获取node.rb（貌似不能用，可以通过all-in-one方式安装后获取）
	[root@puppetmaster162 ~]# cp /usr/share/foreman-installer/modules/foreman/files/foreman-report_v2.rb  /etc/puppet/node.rb

	[root@puppetmaster162 ~]# chown puppet. /etc/puppet/node.rb  #设置属组和属主都为puppet
	[root@puppetmaster162 ~]# chmod 550 /etc/puppet/node.rb  #设置执行权限

**5.3、设置report**

	#从foreman-installer中获取foreman.rb
	[root@puppetmaster162 ~]# cp /usr/share/foreman-installer/modules/foreman/files/foreman-report_v2.rb  /usr/lib/ruby/site_ruby/1.8/puppet/reports/foreman.rb

**5.4、设置连接foreman的信息**

	#这里跟foreman1.5版本（包括1.5版本）不一样，请注意
	[root@puppetmaster162 puppet]# vim /etc/puppet/foreman.yaml 
	---
	:url: "https://foreman163.kisspuppet.com"
	:ssl_ca: "/var/lib/puppet/ssl/certs/ca.pem"
	:ssl_cert: "/var/lib/puppet/ssl/certs/puppetmaster162.kisspuppet.com.pem"
	:ssl_key: "/var/lib/puppet/ssl/private_keys/puppetmaster162.kisspuppet.com.pem"
	:user: ""
	:password: ""
	:puppetdir: "/var/lib/puppet"
	:puppetuser: "puppet"
	:facts: true
	:timeout: 10
	:threads: null
	[root@puppetmaster162 ~]# /etc/init.d/foreman-proxy restart
	Stopping foreman-proxy:                                    [  OK  ]
	Starting foreman-proxy:                                    [  OK  ]

## 6、注册puppet和puppetca ##

**6.1、在puppetmaster上添加ENC配置和foreman报告**

	[root@puppetmaster162 ~]# vim /etc/puppet/puppet.conf 
	[master]
		...
	    reports        = foreman
	    external_nodes = /etc/puppet/node.rb
	    node_terminus  = exec
	#重启生效
	[root@puppetmaster162 ~]# /etc/init.d/puppetmaster restart
	Stopping puppetmaster:                                     [  OK  ]
	Starting puppetmaster:                                     [  OK  ]


**6.2、登录foreman注册foreman-proxy**

![Foreman安装](http://kisspuppet.com/img/foreman04-5.jpg)

**6.3、节点测试**

	[root@foreman163 ~]# puppet  agent -t
	Info: Caching catalog for foreman163.kisspuppet.com
	Info: Applying configuration version '1417762929'
	Notice: Finished catalog run in 0.13 seconds
	
	[root@puppetmaster162 ~]# puppet  agent -t 
	Info: Caching catalog for puppetmaster162.kisspuppet.com
	Info: Applying configuration version '1417762858'
	Notice: Finished catalog run in 0.14 seconds

![Foreman安装](http://kisspuppet.com/img/foreman04-6.jpg)

**注：**如果测试报错，请将foreman中的puppet插件的enc_environment选项设置为false，具体如何使用后续讲解

关于如何设置和使用foreman，请关注后续文章....