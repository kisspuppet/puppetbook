#### Puppet基础篇10-自定义fact实现的四种方式介绍


## 自定义fact可以让节点增加更多的标签 ##

在使用puppet作为配置管理工具的同时，facter是一个非常有用的系统盘点工具，这个工具可以通过一些预先设定好变量定位一台主机，比如可以通过变量lsbdistrelease便可以知道当前系统的版本号，通过osfamily便可以知道系统是RedHat还是SLES，还是其它等等。但是这些预先设定好的变量毕竟有限，在整个自动化运维过程中，由于系统应用的多样性，更多需要通过应用的名称、角色的名称进行标示，这样就需要自定义一些fact并赋值到每个节点上去，相当于给节点打上标签。
<!--more-->

# 一、自定义(custom)fact的四种方法 #

## 1、定义到facter软件本身的lib库中 ##

这种方法是直接在安装facter的lib库里面直接创建，相当于扩充facter软件的lib库。

可以通过以下方法找到facter包的lib库路径为`/usr/lib/ruby/site_ruby/1.8/facter`

	[root@agent1 facter]# rpm -ql facter
	/usr/bin/facter
	/usr/lib/ruby/site_ruby/1.8/facter
	/usr/lib/ruby/site_ruby/1.8/facter.rb
	/usr/lib/ruby/site_ruby/1.8/facter/Cfkey.rb
	/usr/lib/ruby/site_ruby/1.8/facter/application.rb
	/usr/lib/ruby/site_ruby/1.8/facter/architecture.rb
	/usr/lib/ruby/site_ruby/1.8/facter/augeasversion.rb
	/usr/lib/ruby/site_ruby/1.8/facter/blockdevices.rb
	/usr/lib/ruby/site_ruby/1.8/facter/domain.rb
	/usr/lib/ruby/site_ruby/1.8/facter/ec2.rb
	/usr/lib/ruby/site_ruby/1.8/facter/facterversion.rb
	/usr/lib/ruby/site_ruby/1.8/facter/filesystems.rb
	/usr/lib/ruby/site_ruby/1.8/facter/fqdn.rb
	/usr/lib/ruby/site_ruby/1.8/facter/hardwareisa.rb
	/usr/lib/ruby/site_ruby/1.8/facter/hardwaremodel.rb

**1.1、在facter的lib库中创建fact，名称为rpms，可以显示当前安装了多少rpm包**

	[root@agent1 ~]# cd /usr/lib/ruby/site_ruby/1.8/facter/
	[root@agent1 facter]# vim rpms.rb 
	Facter.add(:rpms) do
	  setcode do
	    %x{/bin/rpm -qa | wc -l}.chomp  #定义一个shell命令
	  end
	end

**1.2、通过facter命令进行测试**

	[root@agent1 facter]# facter | grep rpms
	rpms => 918
	[root@agent1 facter]# facter  rpms
	918

**备注：**这种方法相当于给facter软件打补丁，过多的使用可能会破坏facter本身软件的完整性，不建议使用。


## 2、使用环境变量‘FACTERLIB’创建fact ##

这种方法也非常简单，在一个目录下定义一个fact，然后export即可，方法如下

**2.1、在自定义目录里面定义一个fact，列出当前系统登录的用户数**

	[root@agent1 ~]# vim /var/lib/puppet/kiss_fact/users.rb 
	
	Facter.add(:users) do
	  setcode do
	    %x{/usr/bin/who |wc -l}.chomp
	  end
	end
    [root@agent1 kiss_fact]# facter users #无显示结果，需要设置FACTERLIB
    [root@agent1 kiss_fact]#

**2.2、将自定义fact路径赋值给变量FACTERLIB**

	[root@agent1 kiss_fact]# export FACTERLIB=/var/lib/puppet/kiss_fact
	[root@agent1 kiss_fact]# facter users
	2
	[root@agent1 kiss_fact]# facter | grep users
	users => 2

**备注：**这种方法是对第一种方法的扩展，可以自己定义目录，不过需要将路径加到变量FACTERLIB中，可以在/etc/profile添加，这样系统启动的时候便可以自动加载。

## 3、添加外部的(external)fact ##

这种方式支持txt、yaml、json、sh四种格式，内容也比较固定，默认情况下需要在目录/etc/facter/facts.d/下创建，使用也非常方便。
关于这个路径其实是可以修改的，不过修改起来并不是很方便，需要修改facter软件代码。

	[root@agent1 ~]# vim /usr/lib/ruby/site_ruby/1.8/facter/util/config.rb
	 32   def self.external_facts_dirs
	 33     if Facter::Util::Root.root?
	 34       windows_dir = windows_data_dir
	 35       if windows_dir.nil? then
	 36         ["/etc/facter/facts.d", "/etc/puppetlabs/facter/facts.d"]  #external路径位置
	 37       else
	 38         [File.join(windows_dir, 'PuppetLabs', 'facter', 'facts.d')]
	 39       end
	 40     else
	 41       [File.expand_path(File.join("~", ".facter", "facts.d"))]
	 42     end
	 43   end


**特殊说明：**只能用于1.7.3版本以上

**3.1、通过txt文件创建**

**3.1.1、创建roles.txt文件**

文件内容格式必须为“key=value”

	[root@agent1 ~]# mkdir /etc/facter/facts.d -p
	[root@agent1 facts.d]# vim roles.txt 
	web=http
	db=mysql

**3.1.2、测试**

	[root@agent1 facts.d]# facter web1
	http1
	[root@agent1 facts.d]# facter db1
	mysql1
	[root@agent1 facts.d]# facter | egrep 'web1|db1'
	db1 => mysql1
	web1 => http1


**3.2、通过yaml文件创建**

**3.2.1、创建yaml文件**

	[root@agent1 facts.d]# vim roles.yaml 
	---
	
	web2:
	 - http2
	db2:
	 - mysql2
	 - 
**3.2.2、测试**

	[root@agent1 facts.d]# facter | egrep 'web2|db2'
	db2 => mysql2
	web2 => http2

**3.3、通过json文件创建**

**3.3.1、创建json文件**

	[root@agent1 facts.d]# vim roles.json
	{
	   "web3": "http3",
	   "db3": "mysql3"
	}

**备注：**提供一个在线编辑json的网站[http://www.bejson.com/go.html?u=http://www.bejson.com/jsonview2/](http://www.bejson.com/go.html?u=http://www.bejson.com/jsonview2/)

**3.3.2、安装rubygem-json包（json文件需要它的支持）**

	[root@agent1 facts.d]# facter | egrep 'web3|db3'
	Cannot parse JSON data file /etc/facter/facts.d/roles.json without the json library.
	Suggested next step is `gem install json` to install the json library. #缺少json包
	[root@agent1 facts.d]# rpm -ivh rubygem-json-1.5.5-2.el5.x86_64.rpm #安装rubygem-json包，找不到安装包的同志可在群共享里面查找，群号码在文章最后面。

**3.3.3、测试**

	[root@agent1 facts.d]# facter | egrep 'web3|db3'
	db3 => mysql3
	web3 => http3

**3.4、通过sh脚本创建**

**3.4.1、创建shell脚本**

	[root@agent1 facts.d]# vim roles.sh
	#!/bin/bash
	echo "web4=http4"
	echo "db4=mysql4"

**3.4.2、设置文件具有可执行权限**

	[root@agent1 facts.d]# chmod a+x roles.sh 

**3.4.3、测试**

	[root@agent1 facts.d]# facter web4 db4
	db4 => mysql4
	web4 => http4

**思考：**那么如何做到所有节点批量部署呢，可以看到以上四种方式都是基于文件编辑的，可在puppetmaster端通过file资源实现部署。


## 4、使用pluginsync进行发布 ##

这种方法比较特殊，节点factpath目录里除了编写好的rb文件之外，还需要在puppet模块中引用，运行一次之后才会转换成fact。通常在puppetmaster端模块里的lib库中添加，然后在puppet.conf中添加选项pluginsync=true即可，格式为ruby文件。

**4.1、创建模块facts**

	[root@puppetmaster ~]# cd /etc/puppet/environments/kissprd/environment/modules/
	[root@puppetmaster modules]# tree facts/  #目录结构
	facts/
	└── lib
	    └── facter
	        └── hwclock.rb
	
	2 directories, 1 file

**备注：**也可以放在其他已经编写好的模块中

	[root@puppetmaster facter]# vim hwclock.rb  #自定义fact：hwclock，显示节点硬件时间
	Facter.add(:hwclock) do
	  setcode do
	    %x{/usr/sbin/hwclock}.chomp
	  end
	end

**4.2、应用自定义fact至motd模块中**

	[root@puppetmaster kissprd]# vim application/modules/motd/manifests/init.pp 
	
	class motd{
	  package{ 'setup':
	    ensure => present,
	  }
	  file{ '/etc/motd':
	    ensure  => present,
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	    source  => "puppet://$puppetserver/modules/motd/etc/motd",
	    require => Package['setup'],
	  }
	  notify { " Hardware-Clock: ${::hwclock}": } #添加一个通知，这里只是测试，没有实际意义
	}

**4.3、在puppetmaster端的puppet.conf中添加选项pluginsync**

	[root@puppetmaster kissprd]# vim /etc/puppet/puppet.conf
	
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	    pluginsync = true    #添加插件选项
    ...

**4.4、在所有节点puppet.conf中添加pluginsync（通过在puppet模块中添加实现）**

	[root@puppetmaster kissprd]# vim environment/modules/puppet/templates/puppet.conf.erb 
	### config by  puppet ###
	[main]
	    logdir = /var/log/puppet
	    rundir = /var/run/puppet
	    ssldir = $vardir/ssl
	    pluginsync = true  #添加插件选项
	[agent]
	    classfile = $vardir/classes.txt
	    localconfig = $vardir/localconfig
	    server = <%= scope.lookupvar('puppet::params::puppetserver') %>
	    certname = <%= scope.lookupvar('puppet::params::certname') %>

**4.5、节点运行puppet agent进行测试**

	[root@agent1 ~]# facter -p hwclock  #没有这个fact，自定义fact需要加上-p参数才能显示
	[root@agent1 ~]# puppet agent -t --environment=kissprd  #运行一次
	info: Retrieving plugin
	notice: /File[/var/lib/puppet/lib/facter/historys.rb]/ensure: removed
	notice: /File[/var/lib/puppet/lib/facter/hwclock.rb]/ensure: defined content as '{md5}d8cc9fe2b349a06f087692763c878e28'
	info: Loading downloaded plugin /var/lib/puppet/lib/facter/hwclock.rb  #下载插件至节点factpath指定的目录
	info: Loading facts in /var/lib/puppet/lib/facter/hwclock.rb
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1396170375'
	notice:  Hardware-Clock: Sun 30 Mar 2014 05:06:16 PM CST  -0.055086 seconds
	notice: /Stage[main]/Motd/Notify[ Hardware-Clock: Sun 30 Mar 2014 05:06:16 PM CST  -0.055086 seconds]/message: defined 'message' as ' Hardware-Clock: Sun 30 Mar 2014 05:06:16 PM CST  -0.055086 seconds' #应用
	notice: Finished catalog run in 0.51 seconds
	[root@agent1 ~]# facter -p  hwclock #自定义的hwclock生效
	hwclock => Sun 30 Mar 2014 05:06:25 PM CST  -0.567090 seconds

	[root@agent1 ~]# ll /var/lib/puppet/lib/facter/  #插件已经下载到本地
	total 4
	-rw-r--r-- 1 root root 79 Mar 30 17:06 hwclock.rb

关于factpath默认路径可通过以下命令查看，当然也可以在puppet.conf中进行修改

	[root@agent1 ~]# puppet --genconfig | grep factpath
	    factpath = /var/lib/puppet/lib/facter:/var/lib/puppet/facts

## 资料参考： ##

- 官网 [http://docs.puppetlabs.com/guides/custom_facts.html](http://docs.puppetlabs.com/guides/custom_facts.html)

- 书籍《pro puppet 2》Chapter10:Extending Facter and Puppet  **下载地址：**[http://kisspuppet.com/2014/03/29/propuppet2/](http://kisspuppet.com/2014/03/29/propuppet2/)


- 书籍《Puppet 3 Cookbook》Chapter8: External Tools the Puppet Ecosystem **下载地址：**QQ群里已经共享，群号在文章最下面


那么在生产环境当中应当如何应用自定义的fact呢，下一章节会介绍一种方法，结合ENC(hiera)实现节点分类。




