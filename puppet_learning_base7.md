#### Puppet基础篇7-编写第一个完整测试模块puppet


# 工欲善其事必先利其器 #

将Puppet部署到生产中第一个要编写的模块就是puppet本身，虽然puppet可以运行其它所有模块完成各自的部署，但是puppet一旦出问题，那么一切都会停止工作。当然除了puppet自身模块外，还需要保证网络的通畅以及其它你附加的环境等等。

之前编写过简单的motd模块，大致了解了一些模块的结构以及简单的pp语法，接下来我们进行详细的讲解。
那么编写一个完整的puppet模块应该考虑哪些因素呢？

- puppet及附属依赖包是否已经安装OK？

- puppet配置文件是否正确？

- puppet服务是否正常运行？

- 在更新puppet配置文件的情况下，是否能够主动让puppet服务重启或者reload？

- puppet安装包是否能够自动升级到指定版本？

<!--more-->

接下来以agent1和agent3为例进行讲解

## 1、创建puppet模块目录结构 ##

	[root@puppetmaster ~]# cd /etc/puppet/modules/
	[root@puppetmaster modules]# mkdir puppet
	[root@puppetmaster modules]# cd puppet/
	[root@puppetmaster puppet]# mkdir files manifests templates #创建模块目录结构
	[root@puppetmaster puppet]# tree ../puppet
	../puppet
	├── files  #存放下载的文件
	├── manifests  #存放puppet配置
	└── templates  #存放配置模板，方便pp文件引用
	
	3 directories, 0 files
	[root@puppetmaster puppet]# 

## 2、创建puppet配置文件 ##

	[root@puppetmaster puppet]# cd manifests/
	[root@puppetmaster manifests]# touch init.pp config.pp install.pp service.pp params.pp
	[root@puppetmaster manifests]# tree ../
	../
	├── files
	├── manifests
	│   ├── config.pp  #管理puppet配置
	│   ├── init.pp    #管理模块所有pp文件配置
	│   ├── install.pp #管理puppet安装
	│   ├── params.pp  #管理模块中变量以及一些判断
	│   └── service.pp #管理puppet服务
	└── templates
	
	3 directories, 5 files

## 3、编写puppet模块配置文件 ##

整个过程应该是这样，首先应该安装puppet（install.pp），然后配置puppet（config.pp），最后启动puppet服务（service.pp）

**注意：** 接下来的过程不是一步到位的，是一个循序渐进的过程，一步步指导直到完成一个比较完整的模块。

**3.1、编写安装配置文件install.pp**

通过package资源实现，更多有关package的语法及案例请访问[http://kisspuppet.com/2013/11/11/package/](http://kisspuppet.com/2013/11/11/package/)

**需要注意的是：**class名称要和创建的模块名保持一致，名称为puppet，由于在整个配置文件中init.pp为起始配置文件，包含的都应该是子配置文件，所有应该写成“class主类名称::class子类名称”，而class子类名称需要和创建的pp文件名保持一致，比如puppet::install，那么创建的子类名称就应该是install.pp


**3.1.1、编写不具备判断条件的配置文件**

节点安装puppet主要还依赖于facter

	[root@puppetmaster manifests]# vim install.pp 
	class puppet::install{  #一个类包含两个子类
	  include puppet::puppet_install,puppet::facter_install
	}
	class puppet::puppet_install{
	  package { 'puppet':
	    ensure => installed,  #要求处于被安装状态
	  }
	}
	class puppet::facter_install{
	  package { 'facter':
	    ensure => installed,
	  }
	}

也可以用以下两种写法

	[root@puppetmaster manifests]# vim install.pp 
	class puppet::install{  #一个类包含两个资源
	  package { 'puppet':
	    ensure => installed,
	  }
	  package { 'facter':
	    ensure => installed,
	  }
	}

	[root@puppetmaster manifests]# vim install.pp 
	class puppet::install{
	  package { ['puppet','facter']:  #采用数组的形式
	    ensure => installed,
	  }
	}

**3.1.2、编写具备判断系统版本条件的模块**

存在这样一种情况，在我的yum源中有很多puppet版本，而我只希望所有节点只安装我指定的版本，比如2.7.25，那么如何设置呢？其次，还应该考虑一种情况，节点的系统版本可能会不一样，比如有RHEL5、RHEL6等，那么如何让puppet模块自己去判断呢？

通过以下facter进行判断

	[root@agent1 ~]# facter | grep operatingsystemmajrelease
	operatingsystemmajrelease => 5
	[root@agent3 ~]# facter | grep operatingsystemmajrelease
	operatingsystemmajrelease => 6

应该是以下写法比较合理

	[root@puppetmaster manifests]# vim install.pp 
	class puppet::install{
	  include puppet::puppet_install,puppet::facter_install
	}
	class puppet::puppet_install{
	  package { 'puppet':
	    ensure => $operatingsystemmajrelease ?{ #判断系统版本
	      5 => '2.7.25-1.el5',
	      6 => '2.7.25-1.el6',
	    }
	  }
	}
	class puppet::facter_install{
	  package { 'facter':
	    ensure => $operatingsystemmajrelease ?{
	      5 => '1.7.5-1.el5',
	      6 => '1.7.5-1.el6',
	    }
	  }
	}

**3.1.3 添加子类到init.pp中**

	[root@puppetmaster manifests]# vim init.pp 
	class puppet{
	  include puppet::install
	}

**3.1.4 应用到puppet主配置文件site.pp中的节点上**

	[root@puppetmaster ~]# vim /etc/puppet/manifests/site.pp 
	
	$puppetmaster = 'puppetmaster.kisspuppet.com'
	
	node 'puppetmaster_cert.kisspuppet.com'{
	  include  motd,puppet
	}
	node 'agent1_cert.kisspuppet.com'{
	  include  motd,puppet
	}
	
	node 'agent2_cert.kisspuppet.com'{
	  include  motd,puppet
	}
	
	node 'agent3_cert.kisspuppet.com'{
	  include  motd,puppet
	}

也可以是以下写法

	[root@puppetmaster ~]# vim /etc/puppet/manifests/site.pp 
	$puppetmaster = 'puppetmaster.kisspuppet.com'
	
	class environments{
	  include motd,puppet
	}
	
	node 'puppetmaster_cert.kisspuppet.com'{
	  include  environments
	}
	node 'agent1_cert.kisspuppet.com'{
	  include  environments
	}
	
	node 'agent2_cert.kisspuppet.com'{
	  include  environments
	}
	
	node 'agent3_cert.kisspuppet.com'{
	  include  environments
	}

如何所有节点都使用相同的模块，也可以是以下写法

	[root@puppetmaster ~]# vim /etc/puppet/manifests/site.pp 
	
	$puppetmaster = 'puppetmaster.kisspuppet.com'
	
	class environments{
	  include motd,puppet
	}
	
	node default{
	   include environments
	}


**3.1.5、进行简单的测试**

降低facter版本为1.7.3

	[root@agent1 ~]# rpm -e facter --nodeps
	[root@agent1 ~]# rpm -ivh facter-1.7.3-1.el5.x86_64.rpm 
	warning: facter-1.7.3-1.el5.x86_64.rpm: Header V3 RSA/SHA1 signature: NOKEY, key ID 4bd6ec30
	Preparing...                ########################################### [100%]
	   1:facter                 ########################################### [100%]
	[root@agent1 ~]# facter --version
	1.7.3

通过--noop进行尝试性测试，可以看到节点变化情况，但是不进行更改，这也是puppet强大的地方之一

	[root@agent1 ~]# puppet agent -t --noop
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394794815'
	notice: /Stage[main]/Puppet::Facter_install/Package[facter]/ensure: current_value 1.7.3-1.el5, should be 1.7.5-1.el5 (noop)
	notice: Class[Puppet::Facter_install]: Would have triggered 'refresh' from 1 events
	notice: Stage[main]: Would have triggered 'refresh' from 1 events
	notice: Finished catalog run in 0.23 seconds
	[root@agent1 ~]# facter --version
	1.7.3

强制执行，可以看到管理端的facter版本变成了puppet模块中指定的版本1.7.5，这其实也说明了rpm包升级的方法！

	[root@agent1 ~]# puppet agent -t 
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394794815'
	notice: /Stage[main]/Puppet::Facter_install/Package[facter]/ensure: ensure changed '1.7.3-1.el5' to '1.7.5-1.el5'
	notice: Finished catalog run in 6.27 seconds
	[root@agent1 ~]# facter --version
	1.7.5

整个过程是这样，节点同步puppetmaster端后发现facter版本号不对，根据系统类型马上调用底层的安装工具yum（其它系统如suse会调用zypper等）进行安装，整个过程是透明的，而这正是puppet所呈现的强大功能之二。

**3.2、编写配置文件config.pp**

通过file资源实现，更多有关file资源的配置及案例请访问[http://kisspuppet.com/2013/11/14/file/](http://kisspuppet.com/2013/11/14/file/)

**3.2.1、我们暂时只配置puppet.conf文件**

	[root@puppetmaster manifests]# vim config.pp 
	class puppet::config{
	  file { '/etc/puppet/puppet.conf':  #节点文件存放的路径
	    ensure  => present,  #要求存在
	    content => template('puppet/puppet.conf.erb'),  #要求根据模板生成,路径写法为相对路径（templates目录隐藏掉）
	    owner   => 'root',  #要求文件属主为root
	    group   => 'root',  #要求文件属组为root
	    mode    => '0644',  #要求文件权限为644
	    require => Class['puppet::install'],  #要求这个文件在配置之前先正确运行install.pp文件，也就是说要求puppet的包应当处于安装状态
	  }
	}

**3.2.2、编写puppet.conf.erb模板**

puppet的erb模板的存在是为了解决每个节点单独配置一个文件的问题，因为erb模板可以引用fact变量，变量的内容会根据节点系统的不同而变化。

以下为其中一个节点目前的puppet.conf配置文件，我们先找出会变化的内容

    [root@agent1 ~]# vim /etc/puppet/puppet.conf
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com   #变量
	    certname = agent1_cert.kisspuppet.com  #变量
	    runinterval = 10

接下来解决这两个变量

之前我们说过创建params.pp就是为了解决变量问题，我们先用这个解决

找出fact值具有唯一性的fact，比如hostname

	[root@agent1 ~]# facter |grep hostname
	hostname => agent1
	
	[root@agent3 ~]# facter |grep hostname
	hostname => agent3

编写params.pp文件，增加certname变量

	[root@puppetmaster manifests]# vim params.pp 
	class puppet::params {
	  $puppetserver = 'puppetmaster.kisspuppet.com'  #增加puppetserver变量指向puppetmaster名称
	  case $hostname{   #增加certname变量
	    agent1: {
	      $certname = 'agent1_cert.kisspuppet.com'  
	    }
	    agent3: {
	      $certname = 'agent3_cert.kisspuppet.com'
	    }
	    default: {  #设置默认不存在的情况下报错
	      fail("certname is not supported on ${::operatingsystem}")
	    }
	  }
	}

**注意：**这种创建变量的方法在大量节点的情况下显然不是最好的方法，能否通过fact变量实现呢，答案是可以的，可写成以下方式

	[root@puppetmaster manifests]# vim params.pp 
	class puppet::params {
	  $puppetserver = 'puppetmaster.kisspuppet.com'
	  $certname = "${::hostname}_cert.kisspuppet.com"  #通过fact：hostname实现
	  case $operatingsystemmajrelease{
	    5: {
	      $puppet_release = '2.7.23-1.el5'
	      $facter_release = '1.7.3-1.el5'
	    }
	    6: {
	      $puppet_release = '2.7.23-1.el6'
	      $facter_release = '1.7.3-1.el6'
	    }
	    default: {
	      fail("Module puppet is not supported on ${::operatingsystem}")
	    }
	  }
	}

**备注：**这里使用的是默认的fact，如果要通过系统没有的fact应当如何实现呢,后面《Puppet扩展篇1-自定义fact结合ENC(hirea)的应用实践》会有介绍

**思考：**通过变量，尤其是hostname变量确定certname只能保证在hostname不变的情况下才能保证certname不变，否则节点任意修改hostname就会造成certname变更，前期的认证失效，又需要重新认证。那么有没有什么办法解决在hostname变化的情况下certname不变呢，也就是说不需要再次申请证书呢？ 后面《Puppet扩展篇1-自定义fact结合ENC(hirea)的应用实践》会有介绍。

在模板中引用certname变量
注意模板存放的位置要和config.pp中引用模板的位置保持一致

	[root@puppetmaster manifests]# vim ../templates/puppet.conf.erb
    ### config by  puppet ###
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = <%= scope.lookupvar('puppet::params::puppetserver') %>  #引用变量puppetserver
	    certname = <%= scope.lookupvar('puppet::params::certname') %>  #引用变量certname
	    runinterval = 10

由于config.pp依赖于params.pp中的变量，所以config.pp中应当应用class puppet::params


**3.3.3 确定依赖关系**

	[root@puppetmaster manifests]# vim config.pp 
	class puppet::config{
	  include puppet::params   #添加引用关系
	  file { '/etc/puppet/puppet.conf':
	    ensure  => present,
	    content => template('puppet/puppet.conf.erb'),
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	    require => Class['puppet::install'],
	  }
	}

init.pp中应当包含class  puppet::config

	[root@puppetmaster manifests]# vim init.pp 
	class puppet{
	  include puppet::install,puppet::config
	}


**3.3.4  更新测试**

先进行noop测试

	[root@agent1 ~]# puppet agent -t --noop
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394797763'
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: 
	--- /etc/puppet/puppet.conf	2014-03-10 08:22:33.000000000 +0800
	+++ /tmp/puppet-file20140314-7231-f50ehp-0	2014-03-14 19:49:24.000000000 +0800
	@@ -1,3 +1,4 @@
	+### config by  puppet ###  #添加部分
	 [main]
	     logdir = /var/log/puppet
	     rundir = /var/run/puppet
	
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: current_value {md5}fb17740fd53d8d4dfd6d291788a9bda3, should be {md5}134bae34adddbf30a3fe02ff0eb3c6a6 (noop)
	notice: Class[Puppet::Config]: Would have triggered 'refresh' from 1 events
	notice: Stage[main]: Would have triggered 'refresh' from 1 events
	notice: Finished catalog run in 0.43 seconds


强制执行更新

	[root@agent1 ~]# puppet agent -t 
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394797763'
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: 
	--- /etc/puppet/puppet.conf	2014-03-10 08:22:33.000000000 +0800
	+++ /tmp/puppet-file20140314-7475-mlybgg-0	2014-03-14 19:50:16.000000000 +0800
	@@ -1,3 +1,4 @@
	+### config by  puppet ###
	 [main]
	     logdir = /var/log/puppet
	     rundir = /var/run/puppet
	
	info: FileBucket adding {md5}fb17740fd53d8d4dfd6d291788a9bda3
	info: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]: Filebucketed /etc/puppet/puppet.conf to puppet with sum fb17740fd53d8d4dfd6d291788a9bda3
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: content changed '{md5}fb17740fd53d8d4dfd6d291788a9bda3' to '{md5}134bae34adddbf30a3fe02ff0eb3c6a6'
	notice: Finished catalog run in 0.34 seconds
	[root@agent1 ~]# cat /etc/puppet/puppet.conf
	### config by  puppet ###
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com   #根据预先定的puppetserver变量生成
	    certname = agent1_cert.kisspuppet.com  #根据预先定义的certname变量生成
	    runinterval = 10

	[root@agent3 ~]# puppet agent -t
	info: Caching certificate for agent3_cert.kisspuppet.com
	info: Caching certificate_revocation_list for ca
	info: Caching catalog for agent3_cert.kisspuppet.com
	info: Applying configuration version '1394797763'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2010-01-12 21:28:22.000000000 +0800
	+++ /tmp/puppet-file20140314-2786-1wb4mas-0	2014-03-14 19:51:27.589533699 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket adding {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: 
	--- /etc/puppet/puppet.conf	2014-03-09 01:50:46.112175841 +0800
	+++ /tmp/puppet-file20140314-2786-z4e844-0	2014-03-14 19:51:27.719533700 +0800
	@@ -1,3 +1,4 @@
	+### config by  puppet ###
	 [main]
	     logdir = /var/log/puppet
	     rundir = /var/run/puppet
	@@ -8,3 +9,5 @@
	     localconfig = $vardir/localconfig
	     server = puppetmaster.kisspuppet.com
	     certname = agent3_cert.kisspuppet.com
	+    runinterval = 10
	+    
	+    
	+    
	+    
	+     = true
	
	info: FileBucket adding {md5}03cbe6d4def560996eeacedfaef229b4
	info: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]: Filebucketed /etc/puppet/puppet.conf to puppet with sum 03cbe6d4def560996eeacedfaef229b4
	notice: /Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content: content changed '{md5}03cbe6d4def560996eeacedfaef229b4' to '{md5}4f57479998961563e3306b5d0e02a678'
	info: Creating state file /var/lib/puppet/state/state.yaml
	notice: Finished catalog run in 2.86 seconds
	[root@agent3 ~]# cat /etc/puppet/puppet.conf 
	### config by  puppet ###
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = puppetmaster.kisspuppet.com
	    certname = agent3_cert.kisspuppet.com
	    runinterval = 10

**3.3、编写配置文件service.pp**

通过service资源实现，更多有关service资源及案例请访问[http://kisspuppet.com/2013/11/12/service/](http://kisspuppet.com/2013/11/12/service/)

**3.3.1、编写service.pp文件**

	[root@puppetmaster manifests]# vim service.pp 
	class puppet::service{
	  service { 'puppet':
	    ensure     => running,  #设置puppet服务一直处于运行状态
	    hasstatus  => true,  #通过标准的命令“service server_name status"进行检查状态
	    hasrestart => true,  #设置puppet服务具有标准的restart命令
	    enable     => true,  #要求开机自动启动，其实通过chkconfig设置puppet状态为on
	  }
	}

**3.3.2、更新config.pp文件，增加通知服务重启功能**

这个设置完成后，我们再想想我们预先确定的要求是配置在更新后要求puppet服务自动重启，应当做如下设置

	[root@puppetmaster manifests]# vim config.pp 
	class puppet::config{
	  include puppet::params
	  file { '/etc/puppet/puppet.conf':
	    ensure  => present,
	    content => template('puppet/puppet.conf.erb'),
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	    require => Class['puppet::install'],
	    notify  => Class['puppet::service'],  #配置更新后主动通过puppet服务重启
	  }
	}

**3.3.3、添加class puppet::service到init.pp中**

	[root@puppetmaster manifests]# vim init.pp 
	class puppet{
	  include puppet::install,puppet::config,puppet::service
	}

**3.3.4、测试**

测试一：查看是否设置了开机启动，查看puppet服务状态

	[root@agent1 ~]# chkconfig puppet off
	[root@agent1 ~]# /etc/init.d/puppet status
	puppetd is stopped
	
	[root@agent1 ~]# puppet agent -t
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394798692'
	notice: /Stage[main]/Puppet::Service/Service[puppet]/ensure: ensure changed 'stopped' to 'running'
	notice: Finished catalog run in 1.42 seconds
	
	[root@agent1 ~]# chkconfig --list | grep puppet
	puppet         	0:off	1:off	2:on	3:on	4:on	5:on	6:off
	[root@agent1 ~]# /etc/init.d/puppet status
	puppetd (pid  8537) is running...

测试二、查看配置被更改还原后，服务是否会自动重启

	[root@agent1 ~]# echo "#add a line" >>/etc/puppet/puppet.conf
	[root@agent1 ~]# tailf  /var/log/messages
	Mar 14 21:18:52 agent1 puppet-agent[10803]: Reopening log files
	Mar 14 21:18:52 agent1 puppet-agent[10803]: Starting Puppet client version 2.7.25
	Mar 14 21:18:53 agent1 puppet-agent[10803]: Finished catalog run in 0.27 seconds
	Mar 14 21:19:05 agent1 puppet-agent[10803]: Finished catalog run in 0.35 seconds
	Mar 14 21:19:16 agent1 puppet-agent[10803]: Finished catalog run in 0.71 seconds
	Mar 14 21:19:27 agent1 puppet-agent[10803]: Finished catalog run in 0.30 seconds
	Mar 14 21:19:38 agent1 puppet-agent[10803]: Finished catalog run in 0.37 seconds
	Mar 14 21:19:50 agent1 puppet-agent[10803]: Finished catalog run in 0.42 seconds
	Mar 14 21:20:01 agent1 puppet-agent[10803]: Finished catalog run in 0.28 seconds
	Mar 14 21:20:12 agent1 puppet-agent[10803]: Finished catalog run in 0.36 seconds
	Mar 14 21:20:23 agent1 puppet-agent[10803]: Finished catalog run in 0.27 seconds
	Mar 14 21:20:34 agent1 puppet-agent[10803]: (/Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content) content changed '{md5}898865b650b9af4cae1886a894ce656e' to '{md5}8c67cb8c039bb6436556b91f0c6678c4'
	Mar 14 21:20:34 agent1 puppet-agent[10803]: Caught TERM; calling stop
	Mar 14 21:20:36 agent1 puppet-agent[13068]: Reopening log files
	Mar 14 21:20:36 agent1 puppet-agent[13068]: Starting Puppet client version 2.7.25  #重启服务

**3.3.5、服务设置reload动作**

在有些场合，我们仅仅需要在修改配置后，让服务重新reload而不是restart，这又当如何设置呢

	[root@puppetmaster manifests]# vim config.pp 
		class puppet::service{
		  service { 'puppet':
		    ensure     => running, 
		    hasstatus  => true, 
		    hasrestart => true, 
		    enable     => true, 
		    provider   => init,  
		    path       => "/etc/init.d",  #设置启动脚本的搜索路径
		    restart    => "/etc/init.d/sshd reload",  #将restart改成reload
		    start      => "/etc/init.d/sshd start",
		    stop       => "/etc/init.d/sshd stop",
		  }
		}

测试可以看出服务并没有停止，而是refresh了

	[root@agent1 ~]# echo "#add a line" >>/etc/puppet/puppet.conf 
	[root@agent1 ~]# tailf  /var/log/messages
	Mar 14 21:32:03 agent1 puppet-agent[13068]: Finished catalog run in 0.33 seconds
	Mar 14 21:32:13 agent1 puppet-agent[13068]: Reparsing /etc/puppet/puppet.conf
	Mar 14 21:32:14 agent1 puppet-agent[13068]: (/Stage[main]/Puppet::Config/File[/etc/puppet/puppet.conf]/content) content changed '{md5}898865b650b9af4cae1886a894ce656e' to '{md5}8c67cb8c039bb6436556b91f0c6678c4'
	Mar 14 21:32:14 agent1 puppet-agent[13068]: (/Service[puppet]) Triggered 'refresh' from 1 events
	Mar 14 21:32:14 agent1 puppet-agent[13068]: Finished catalog run in 0.32 seconds
	Mar 14 21:32:25 agent1 puppet-agent[13068]: Finished catalog run in 0.25 seconds
	Mar 14 21:32:35 agent1 puppet-agent[13068]: Reparsing /etc/puppet/puppet.conf
	Mar 14 21:32:36 agent1 puppet-agent[13068]: Finished catalog run in 0.25 seconds


**4、优化代码**

**4.1、 将install.pp中的判断语句添加到params.pp中**

	[root@puppetmaster manifests]# vim params.pp 
	
	class puppet::params {
	  $puppetserver = 'puppetmaster.kisspuppet.com'
	  case $hostname{
	    agent1: {
	      $certname = 'agent1_cert.kisspuppet.com'
	    }
	    agent3: {
	      $certname = 'agent3_cert.kisspuppet.com'
	    }
	    default: {
	      fail("certname is not supported on ${::operatingsystem}")
	    }
	  }
	
	  case $operatingsystemmajrelease{   #添加系统版本变量
	    5: {
	      $puppet_release = '2.7.23-1.el5'
	      $facter_release = '1.7.3-1.el5'
	    }
	    6: {
	      $puppet_release = '2.7.23-1.el6'
	      $facter_release = '1.7.3-1.el6'
	    }
	    default: {
	      fail("Module puppet is not supported on ${::operatingsystem}")
	    }
	  }
	}


	[root@puppetmaster manifests]# vim install.pp  #通过变量引用
	class puppet::install{
	  include puppet::puppet_install,puppet::facter_install
	}
	class puppet::puppet_install{
	  package { 'puppet':
	    ensure => $puppet::params::puppet_release,  #puppet里引用变量的方法为“$class::子class::变量”
	  }
	}
	class puppet::facter_install{
	  package { 'facter':
	    ensure => $puppet::params::facter_release,
	  }
	}
         
**4.2、测试（略）**

技术源于分享.....

ssh模块案例详解：[http://dreamfire.blog.51cto.com/418026/1257719](http://dreamfire.blog.51cto.com/418026/1257719)


