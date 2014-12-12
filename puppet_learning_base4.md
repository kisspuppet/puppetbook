#### Puppet基础篇4-安装、配置并使用Puppet


Puppet前期环境（网络、解析、yum源、NTP）在上一章节已经准备就绪，接下来我们就开始安装Puppet了，安装Puppet其实很简单，官方已经提供了yum源，只需要自己将所需要的安装包下载下来然后做成本地yum源即可使用。
**注意：**本实验完全采用自定义的certname名，如果不设置默认会使用系统变量hostname的值。
<!--more-->

## 一、安装Puppetmaster ##
**1、安装Puppet-server、puppet和facter**

	[root@puppetmaster ~]# yum install puppet puppet-server facter -y #系统会自己安装一些ruby依赖包环境

**2、配置puppet.conf**
**注意：**这个里面配置了两个certname名称，其中[master]中配置的certname是为所有节点认证用的master名称，[agent]中配置的certname是他本身agent的名称，当然不配置默认是和master的名称是一样的。

    [root@puppetmaster ~]# cp /etc/puppet/puppet.conf{,.bak}   #备份
	[root@puppetmaster ~]# vim /etc/puppet/puppet.conf  #注释已经删除
	[main]
	    logdir = /var/log/puppet  #默认日志存放路径
	    rundir = /var/run/puppet  #pid存放路径
	    ssldir = $vardir/ssl #证书存放目录，默认$vardir为/var/lib/puppet
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com #设置agent认证连接master端的服务器名称，注意这个名字必须能够被节点解析
	    certname = puppetmaster_cert.kisspuppet.com #设置agent端certname名称
	[master]
	    certname = puppetmaster.kisspuppet.com  puppetmaster.kisspuppet.com #设置puppetmaster认证服务器名

**3、创建site.pp文件**
site.pp文件是puppet读取所有模块pp文件的开始，在3.0版本以前必须设置，否则服务无法启动。

	[root@puppetmaster ~]# touch /etc/puppet/manifests/site.pp

**4、启动puppetmaster服务**

	[root@puppetmaster ~]# /etc/init.d/puppetmaster start
	Starting puppetmaster:       
                              [  OK  ]
	[root@puppetmaster ~]# chkconfig puppetmaster on #设置开机启动

**5、查看本地证书情况**
puppetmaster第一次启动会自动生成证书自动注册自己

	[root@puppetmaster ~]# tree /var/lib/puppet/ssl/
	/var/lib/puppet/ssl/
	├── ca
	│   ├── ca_crl.pem
	│   ├── ca_crt.pem
	│   ├── ca_key.pem
	│   ├── ca_pub.pem
	│   ├── inventory.txt
	│   ├── private
	│   │   └── ca.pass
	│   ├── requests
	│   ├── serial
	│   └── signed
	│       └── puppetmaster.kisspuppet.com.pem  #已注册
	├── certificate_requests
	├── certs
	│   ├── ca.pem
	│   └── puppetmaster.kisspuppet.com.pem
	├── crl.pem
	├── private
	├── private_keys
	│   └── puppetmaster.kisspuppet.com.pem
	└── public_keys
	    └── puppetmaster.kisspuppet.com.pem
	
	9 directories, 13 files
	[root@puppetmaster ~]# puppet cert --list --all  #带+标示已经注册成功
	+ "puppetmaster.kisspuppet.com" (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")

**6、查看监听状态**
puppetmaster服务开启后，默认监听TCP 8140端口

	[root@puppetmaster ~]# netstat -nlatp | grep 8140
	tcp        0      0 0.0.0.0:8140                0.0.0.0:*                   LISTEN      1976/ruby           
	[root@puppetmaster ~]# lsof -i:8140
	COMMAND    PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
	puppetmas 1976 puppet    5u  IPv4  14331      0t0  TCP *:8140 (LISTEN)


## 二、安装Agent ##
以agent1为例

**1、安装puppet和facter**

	[root@agent1 ~]# yum install puppet facter #系统会自己安装一些ruby依赖包环境

**2、配置puppet.conf**

	[root@agent1 ~]# cp /etc/puppet/puppet.conf{,.bak}
	[root@agent1 ~]# vim /etc/puppet/puppet.conf
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	    
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com  #指向puppetmaster端
	    certname = agent1_cert.kisspuppet.com #设置自己的certname名

**3、通过调试模式启动节点向Puppetmaster端发起认证**

	[root@agent1 ~]# puppet agent --test
	info: Creating a new SSL key for agent1_cert.kisspuppet.com
	info: Caching certificate for ca
	info: Creating a new SSL certificate request for agent1_cert.kisspuppet.com
	info: Certificate Request fingerprint (md5): 69:D2:86:E4:7F:00:E0:55:61:19:02:34:9E:9B:AF:F9
	Exiting; no certificate found and waitforcert is disabled


**4、服务器端确定认证**

	[root@puppetmaster ~]# puppet cert --list --all #查看认证情况
	  "agent1_cert.kisspuppet.com"  (69:D2:86:E4:7F:00:E0:55:61:19:02:34:9E:9B:AF:F9) #未认证
	+ "puppetmaster.kisspuppet.com" (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")
	
	[root@puppetmaster ~]# puppet cert --sign agent1_cert.kisspuppet.com #注册agent1
	notice: Signed certificate request for agent1_cert.kisspuppet.com
	notice: Removing file Puppet::SSL::CertificateRequest agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/agent1_cert.kisspuppet.com.pem'

	[root@puppetmaster ~]# puppet cert --list --all #再次查看认证情况
	+ "agent1_cert.kisspuppet.com"  (3E:46:4E:75:34:9A:5A:62:A6:3C:AE:BD:49:EE:C0:F5)
	+ "puppetmaster.kisspuppet.com" (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")

	[root@puppetmaster ~]# tree /var/lib/puppet/ssl/ #另外一种查看认证的方式
	/var/lib/puppet/ssl/
	├── ca
	│   ├── ca_crl.pem
	│   ├── ca_crt.pem
	│   ├── ca_key.pem
	│   ├── ca_pub.pem
	│   ├── inventory.txt
	│   ├── private
	│   │   └── ca.pass
	│   ├── requests
	│   ├── serial
	│   └── signed
	│       ├── agent1_cert.kisspuppet.com.pem  #已经注册成功
	│       └── puppetmaster.kisspuppet.com.pem
	├── certificate_requests
	├── certs
	│   ├── ca.pem
	│   └── puppetmaster.kisspuppet.com.pem
	├── crl.pem
	├── private
	├── private_keys
	│   └── puppetmaster.kisspuppet.com.pem
	└── public_keys
	    └── puppetmaster.kisspuppet.com.pem
	
	9 directories, 14 files

**5、其它节点一起认证**

	[root@puppetmaster ~]# puppet agent --test #puppetmaster自己申请agent认证
	info: Creating a new SSL key for puppetmaster_cert.kisspuppet.com
	info: Creating a new SSL certificate request for puppetmaster_cert.kisspuppet.com
	info: Certificate Request fingerprint (md5): 7D:AC:F7:97:04:2B:E4:C5:74:4A:16:05:DB:F6:6A:98
	Exiting; no certificate found and waitforcert is disabled

	[root@puppetmaster ~]# puppet cert --sign --all #注册所有请求的节点
	notice: Signed certificate request for puppetmaster_cert.kisspuppet.com
	notice: Removing file Puppet::SSL::CertificateRequest puppetmaster_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/puppetmaster_cert.kisspuppet.com.pem'
	notice: Signed certificate request for agent2_cert.kisspuppet.com
	notice: Removing file Puppet::SSL::CertificateRequest agent2_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/agent2_cert.kisspuppet.com.pem'
	notice: Signed certificate request for agent3_cert.kisspuppet.com
	notice: Removing file Puppet::SSL::CertificateRequest agent3_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/agent3_cert.kisspuppet.com.pem'

	[root@puppetmaster ~]# puppet cert --list --all #查看所有节点认证
	+ "agent1_cert.kisspuppet.com"       (3E:46:4E:75:34:9A:5A:62:A6:3C:AE:BD:49:EE:C0:F5)
	+ "agent2_cert.kisspuppet.com"       (A0:CE:70:BE:A9:11:BF:F4:C8:EF:25:8E:C2:2C:3B:B7)
	+ "agent3_cert.kisspuppet.com"       (98:93:F7:0C:ED:94:81:3D:51:14:86:68:2B:F3:F1:A0)
	+ "puppetmaster.kisspuppet.com"      (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")
	+ "puppetmaster_cert.kisspuppet.com" (57:A3:D7:3D:64:2F:D6:FD:BC:2A:6C:79:68:73:EA:AB)

## 三、编写简单的motd模块 ##
**1、创建模块目录结构**
**注意：**再未指定modulepath搜索路径的情况下，会有默认搜索路径的，可通过以下方式查看到

	[root@puppetmaster ~]# puppet master --genconfig >/etc/puppet/puppet.conf.out
	[root@puppetmaster ~]# cat /etc/puppet/puppet.conf.out | grep modulepath
	    modulepath = /etc/puppet/modules:/usr/share/puppet/modules
	
    [root@puppetmaster modules]# tree /etc/puppet/modules/
	/etc/puppet/modules/
	└── motd
	    ├── files  #存放文件目录
	    │   └── etc
	    │       └── motd
	    ├── manifests  #存放模块pp配置文件目录
	    │   └── init.pp
	    └── templates #存放模板目录

	5 directories, 2 files

**2、编写pp文件**

	[root@puppetmaster modules]# vim motd/manifests/init.pp 
	class motd{                 #定义一个类叫motd
	  package{ 'setup':    #定义package资源
	    ensure => present,  #要求setup这个包处于被安装状态
	  }
	  file{ '/etc/motd':  #定义file资源
	    ensure  => present,  #要求file文件处于存在状态
	    owner   => 'root', #要求file文件属主为root
	    group   => 'root', #要求file文件属组为root
	    mode    => '0644', #要求file文件权限为644
	    source  => "puppet://$puppetserver/modules/motd/etc/motd", #要求file文件从puppetmaster端服务器下载
	    require => Package['setup'], #要求文件被配置之前先执行package资源
	  }
	}

	[root@puppetmaster modules]# cat motd/files/etc/motd 
	--                       --
	--------puppet test---------
	--                       --

**3、编写site.pp文件**

	[root@puppetmaster ~]# vim /etc/puppet/manifests/site.pp 
	
	$puppetserver = 'puppetmaster.kisspuppet.com' #设置全局变量
	node 'puppetmaster_cert.kisspuppet.com'{
	  include  motd
	}
	node 'agent1_cert.kisspuppet.com'{
	  include  motd
	}
	
	node 'agent2_cert.kisspuppet.com'{
	  include  motd
	}
	
	node 'agent3_cert.kisspuppet.com'{
	  include  motd
	}


## 四、测试motd模块 ##

	[root@agent1 ~]# puppet agent --test  #测试节点agent1
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394304542'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2000-01-13 07:18:52.000000000 +0800
	+++ /tmp/puppet-file20140309-4571-1vqc18j-0	2014-03-09 02:51:47.000000000 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket adding {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	notice: Finished catalog run in 0.40 seconds
	
    [root@agent1 ~]# cat /etc/motd 
	--                       --
	--------puppet test---------
	--                       --
	[root@agent1 ~]# 

 
	[root@puppetmaster ~]# puppet agent -t  #测试节点puppetmaster
	info: Caching catalog for puppetmaster_cert.kisspuppet.com
	info: Applying configuration version '1394305371'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2010-01-12 21:28:22.000000000 +0800
	+++ /tmp/puppet-file20140309-3102-1gadon0-0	2014-03-09 03:02:51.966998294 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket adding {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	info: Creating state file /var/lib/puppet/state/state.yaml
	notice: Finished catalog run in 0.52 seconds
	[root@puppetmaster ~]# cat /etc/motd 
	--                       --
	--------puppet test---------
	--                       --


