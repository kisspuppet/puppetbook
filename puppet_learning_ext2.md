#### Puppet扩展篇2-如何使用虚拟资源解决puppet冲突问题


虚拟资源是一种用来管理多种配置共同依赖同一资源的方法。如果多个类依赖同一个资源时则可避免写多个资源，也可以解决资源重定义的错误。
虚拟资源经常用于用户管理中，虚拟资源只会被声明一次，但可以运用一次或多次。
<!--more-->

要使用虚拟资源是需要在资源声明开头加上字符“@”来使资源虚拟化。然后再使用下面两种方法之一来实例化虚拟资源：

- "飞船"语法<||>
- realize函数


# 1. 定义两个用户，puppet和root，并将其虚拟化 #

注意定义虚拟资源必须在全局作用域或者节点作用域中定义，简单的理解，以下目录中site.pp就是全局作用域，包含nodes目录（site.pp中import了nodes目录），在节点node下定义的虚拟资源属于节点作用域，其他模块中的都属于局部作用域。

## 1.1 在全局作用域中创建对应的pp文件 ##

	[root@linuxmaster1poc testing]# tree manifests/
	manifests/
	├── nodes
	│   ├── puppetclient.pp
	│   ├── virtual_group.pp
	│   └── virtual_user.pp
	└── site.pp
	
	1 directory, 4 files

----
## 1.2 创建虚拟用户puppet、root、xiaonuo ##

	[root@linuxmaster1poc testing]# vim manifests/nodes/virtual_user.pp 
	class alluser{
	  include alluser::puppet,alluser::root
	}
	
	class alluser::puppet{
	  @user { 'puppet':
	    ensure => present,
	    uid    => '52',
	    gid    => '52',
	    home   => '/var/lib/puppet',
	    shell  => '/sbin/nologin',
	  }
	}
	
	class alluser::root{
	  @user { 'root':
	    ensure => present,
	    uid    => '0',
	    gid    => '0',
	    home   => '/root',
	    shell  => '/bin/bash',
	  }
	}

	class alluser::xiaonuo{
	  @user { 'xiaonuo':
	    ensure => present,
	    uid    => '600',
	    gid    => '600',
	    home   => '/home/xiaonuo',
	    shell  => '/sbin/nologin',
	  }
	}

-------
## 1.3 创建虚拟组puppet、root和xiaonuo ##

	[root@linuxmaster1poc testing]# vim manifests/nodes/virtual_group.pp 
	class allgroup{
	  include allgroup::puppet,allgroup::root
	}
	
	class allgroup::puppet{
	  @group { 'puppet':
	    ensure    => present,
	    name      => 'puppet',
	    gid       => '52',
	    allowdupe => false,
	    members   => 'puppet',
	  }
	}
	
	class allgroup::root{
	  @group { 'root':
	    ensure    => present,
	    name      => 'root',
	    gid       => '0',
	    allowdupe => false,
	    members   => 'root',
	  }
	}
	class allgroup::xiaonuo{
	  @group { 'xiaonuo':
	    ensure    => present,
	    name      => 'xiaonuo',
	    gid       => '600',
	    allowdupe => false,
	    members   => 'xiaonuo',
	  }
	}

# 2. 编写puppet模块，将虚拟资源用户puppet和组puppet实化 #

## 2.1 编写pupppet模块 ##
	[root@linuxmaster1poc testing]# tree environment/modules/puppet
	environment/modules/puppet
	├── files
	├── manifests
	│   ├── config.pp
	│   ├── init.pp
	│   ├── install.pp
	│   ├── params.pp
	│   └── service.pp
	├── README
	└── templates
	    ├── auth.conf.erb
	    ├── namespaceauth.conf.erb
	    └── puppet.conf.erb
	
	3 directories, 9 files

## 2.2 编写puppet_linux57poc模块 ##

	[root@linuxmaster1poc testing]# tree agents/modules/puppet_linux57poc/
	agents/modules/puppet_linux57poc/
	├── files
	├── manifests
	│   └── init.pp
	└── templates
	    ├── facts.txt.erb
	    └── motd.erb
	
	3 directories, 3 files

## 2.3 实例化虚拟资源 ##

**2.3.1 在puppet模块中实例化**

	[root@linuxmaster1poc testing]# vim environment/modules/puppet/manifests/config.pp 
	class puppet::config{
	  include puppet::params
	  include puppet::puppet_config,puppet::namespaceauth_config,puppet::auth_config,puppet::user,puppet::group
	  include alluser,allgroup #必须将节点作用域中的类包含进来
	}
	
	class puppet::puppet_config{
	  file { '/etc/puppet/puppet.conf':
	    ensure  => present,
	    content => template('puppet/puppet.conf.erb'),
	    owner   => 'puppet',
	    group   => 'puppet',
	    mode    => '0644',
	    backup  => main,
	    require => Class['puppet::install','puppet::user','puppet::group'],
	    notify  => Class['puppet::service'],
	  }
	}
	
	
	class puppet::auth_config{
	  file { '/etc/puppet/auth.conf':
	    ensure  => present,
	    content => template('puppet/auth.conf.erb'),
	    owner   => 'puppet',
	    group   => 'puppet',
	    mode    => '0644',
	    backup  => main,
	    require => Class['puppet::install','puppet::user','puppet::group'],
	    notify  => Class['puppet::service'],
	  }
	}
	
	class puppet::namespaceauth_config{
	  file { '/etc/puppet/namespaceauth.conf':
	    ensure  => present,
	    content => template('puppet/namespaceauth.conf.erb'),
	    owner   => 'puppet',
	    group   => 'puppet',
	    mode    => '0644',
	    backup  => main,
	    require => Class['puppet::install','puppet::user','puppet::group'],
	    notify  => Class['puppet::service'],
	  }
	}
	
	class puppet::user{ #使用飞船语法实化用户puppet资源
	#  realize User['puppet'] 
	  User <| title == 'puppet' |>
	}
	
	class puppet::group{ #使用realize函数实化组puppet资源
	  realize Group['puppet']
	#  Group <| title == 'puppet' |>
	}

**2.3.2 在puppet_linux57poc模块中实例化**

	[root@linuxmaster1poc testing]# cat agents/modules/puppet_linux57poc/manifests/init.pp 
	class puppet_linux57poc{
	  include puppet_linux57poc::motd_install,puppet_linux57poc::motd_config,puppet_linux57poc::facts,puppet_linux57poc::user,puppet_linux57poc::group
	  include alluser,allgroup #必须将节点作用域中的类包含进来
	}
	
	class puppet_linux57poc::motd_install{
	  package{ setup:
	    ensure => present,
	  }
	}
	class puppet_linux57poc::motd_config{
	  file{ "/etc/motd":
	    owner   => "xiaonuo",
	    group   => "root",
	    mode    => 0440,
	    content => template("puppet_linux57poc/motd.erb"),
	    backup  => 'main',
	    require => Class['puppet_linux57poc::motd_install','puppet_linux57poc::user','puppet_linux57poc::group']
	  }
	}
	class puppet_linux57poc::facts{
	  file{ "/etc/mcollective/facts.txt":
	    owner   => "root",
	    group   => "root",
	    mode    => 0400,
	    content => template("puppet_linux57poc/facts.txt.erb"),
	    backup  => 'main',
	    require => Class['puppet_linux57poc::motd_install','puppet_linux57poc::user','puppet_linux57poc::group']
	  }
	}
	
	class puppet_linux57poc::user{  #使用realize函数实化用户xiaonuo和root资源
	  realize( User['xiaonuo'], 
	           User['root'] )
	}
	
	class puppet_linux57poc::group{ #使用realize函数实化组xiaonuo和root资源
	  realize( Group['xiaonuo'],
	           Group['root'] )
	}

# 3. 测试 #

## 3.1 测试puppet模块（略） ##

## 3.2 测试puppet_linux57poc模块 ##

**3.2.1 查看当前系统是否有xiaonuo用户和组**

	[root@linux57poc puppet]# id xiaonuo
	id: xiaonuo: No such user
	[root@linux57poc puppet]# cat /etc/group | grep xiaonuo
	[root@linux57poc puppet]# 
	[root@linux57poc puppet]# ll /etc/motd 
	-rwxrwxrwx 1 puppet puppet 313 Jan  2 06:17 /etc/motd

**3.2.2 同步puppetmaster**

	[root@linux57poc puppet]# puppet agent -t --environment=testing
	info: Retrieving plugin
	info: Loading facts in /var/lib/puppet/lib/facter/fact_apply.rb
	info: Caching catalog for puppet_linux57poc.dev.shanghaigm.com
	info: Applying configuration version '1389555288'
	notice: /Stage[main]/Allservice::Lm_sensors_service/Service[lm_sensors]/ensure: ensure changed 'running' to 'stopped'
	notice: /Group[xiaonuo]/ensure: created
	notice: /Stage[main]/Alluser::Xiaonuo/User[xiaonuo]/ensure: created
	...
	info: FileBucket adding {md5}b2090646c444c5ddf1533749743ebd71
	info: /Stage[main]/Mcollective::Facter/File[/etc/mcollective/facts.yaml]: Filebucketed /etc/mcollective/facts.yaml to main with sum b2090646c444c5ddf1533749743ebd71
	notice: /Stage[main]/Sysctl::Exec/Exec[sysctl -p >/dev/null &]/returns: executed successfully
	notice: /Stage[main]/Puppet_linux57poc::Motd_config/File[/etc/motd]/owner: owner changed 'puppet' to 'xiaonuo'
	notice: /Stage[main]/Puppet_linux57poc::Motd_config/File[/etc/motd]/group: group changed 'puppet' to 'root'
	notice: /Stage[main]/Puppet_linux57poc::Motd_config/File[/etc/motd]/mode: mode changed '0777' to '0440'
	notice: /Stage[main]/Allservice::Bluetooth_service/Service[bluetooth]/ensure: ensure changed 'running' to 'stopped'
	notice: Finished catalog run in 4.54 seconds

**3.2.3 验证结果是否正确**

	[root@linux57poc puppet]# id xiaonuo
	uid=600(xiaonuo) gid=600(xiaonuo) groups=600(xiaonuo)
	[root@linux57poc puppet]# cat /etc/group | grep xiaonuo
	xiaonuo:x:600:
	[root@linux57poc puppet]# ll /etc/motd 
	-r--r----- 1 xiaonuo root 313 Jan  2 06:17 /etc/motd
	[root@linux57poc puppet]# 


