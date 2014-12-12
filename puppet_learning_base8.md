#### Puppet基础篇8-编写第二个完整测试模块yum


# 工欲善其事必先利其器 #

上一节讲解了puppet基础环境模块puppet，除此之外影响puppet基础环境的还有一个模块叫yum源，当然这个是相对于RedHat系统而言的，如果是SLES系统，就要配置zypper源了，其它Linux系统也是如此。那么配置yum源需要用到哪些资源呢？

之前写puppet模块的时候用到了file资源、service资源、package资源，那么这三个资源是否能满足yum模块的配置呢，答案是肯定的。然而官方给出了专用的yumrepo资源，管理可以精确到repo里面的每一行，使用还是非常方便的，接下来，我们使用官方给出的yumrepo资源来配置yum模块。

**注：**上一节教会大家如何一步步创建一个完整模块，为了避免重复，这一节就直接贴配置了。

<!--more-->

## 一、配置之前需要考虑的问题： ##

1、yum包需要被安装；

2、yum主配置文件yum.conf需要配置正确；

3、每台主机至少有两个repo源，一个指向本地的ISO源，一个指向自定义的puppet源；

4、不同系统版本的repo源中的部分参数略有不同，比如baseurl。

## 二、创建yum模块 ##

**1、创建yum模块目录结构**

	[root@puppetmaster modules]# tree yum
	yum
	├── files
	├── manifests
	└── templates
	
	3 directories, 0 files

**2、创建package资源**

	[root@puppetmaster manifests]# vim install.pp
	
	class yum::install{
	  package { 'yum':
	    ensure => installed, #要求yum这个包处于安装状态
	  }
	}

**3、创建params.pp**

根据操作系统版本定义repo文件中的各项条目

	eg.
	[root@agent1 ~]# facter | grep operatingsystemrelease   系统版本fact
	operatingsystemrelease => 5.7

由于RedHat存在多个版本，不同版本yum源的指向不同，对应的pki认证文件也不同，因此应当设置一些变量，然后进行引用。以下只定义了系统版本为5.7、5.8、和6.4的变量，如果有其它版本效仿即可。

	[root@puppetmaster manifests]# vim params.pp
	class yum::params {
	  case $operatingsystemrelease{
	    5.7: {
	      $yum_redhat_descr = 'rhel base rpm packages'  #定义redhat光盘源的描述信息
	      $yum_puppet_descr = 'puppet rpm packages for rhel' #定义puppet源的描述信息
	      $yum_redhat_pki = 'file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel5' #定义redhat光盘源的pki认证文件位置
	      $yum_redhat_baseurl = 'file:///media/cdrom/Server'  #定义redhat光盘源baseurl的下载位置
	      $yum_puppet_baseurl = 'ftp://puppetmaster.kisspuppet.com/RHEL5U7' #定义puppet源baseurl的下载位置
	      $yum_redhat_pki_name = '/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel5' #定义puppet源pki认证文件位置
	      $yum_redhat_pki_download = 'puppet:///modules/yum/PM-GPG-KEY/RPM-GPG-KEY-redhat-release-rhel5' #定义pki文件的服务器下载地址
	    }
			
	    5.8: {
	      $yum_redhat_descr = 'rhel base rpm packages'
	      $yum_puppet_descr = 'puppet rpm packages for rhel'
	      $yum_redhat_pki = 'file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel5'
	      $yum_redhat_baseurl = 'file:///media/cdrom/Server'
	      $yum_puppet_baseurl = 'ftp://puppetmaster.kisspuppet.com/RHEL5U8'
	      $yum_redhat_pki_name = '/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel5'
	      $yum_redhat_pki_download = 'puppet:///modules/yum/PM-GPG-KEY/RPM-GPG-KEY-redhat-release-rhel5'
	    }
					
	    6.4: {
	      $yum_redhat_descr = 'rhel base rpm packages'
	      $yum_puppet_descr = 'puppet rpm packages for rhel'
	      $yum_redhat_pki = 'file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel6'
	      $yum_redhat_baseurl = 'file:///media/cdrom'
	      $yum_puppet_baseurl = 'ftp://puppetmaster.kisspuppet.com/RHEL6U4'
	      $yum_redhat_pki_name = '/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel6'
	      $yum_redhat_pki_download = 'puppet:///modules/yum/PM-GPG-KEY/RPM-GPG-KEY-redhat-release-rhel6'
	    }
	   default: {  #定义如果没有以上版本的系统，直接报以下错误，同时也是为了方便调试
	      fail("Module yum is not supported on ${::operatingsystem}")
	    }
	  }
	}

**4、创建config.pp文件**

config.pp文件用于管理yum主配置文件yum.conf，repo文件的属性，pki文件的属性及下载地址和yumrepo源

	[root@puppetmaster manifests]# vim config.pp
	class yum::config{
	  include yum::params  #引用class yum::params
	  include yum::config_file,yum::config_key,yum::config_repo
	}
	
	class yum::config_file{
	  file { '/etc/yum.conf':  #创建file资源管理yum主配置文件yum.conf
	    ensure  => present, #要求文件处于存在状态
	    owner   => 'root',  #属主为root
	    group   => 'root',  #属组为root
	    mode    => '0644',  #文件权限为644
	    source  => 'puppet:///modules/yum/etc/yum.conf', #要求从puppetmaster服务器指定目录去下载
	    require => Class['yum::install'], #要求在配置之前先安装yum软件包
	  }
	  file { '/etc/yum.repos.d/rhel-base.repo':  #设置光盘repo的一些属性
	    ensure  => present,
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	    require => Class['yum::config_repo'], #要求设置之前yumrepo资源rhel-base必须存在
	  }
	  file { '/etc/yum.repos.d/rhel-puppet.repo':  #设置puppet repo的一些属性
	    ensure  => present,
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	    require => Class['yum::config_repo'],  #要求设置之前yumrepo资源puppet必须存在
	  }
	}
	class yum::config_key{  #设置pki证书的一些属性及下载位置
	  file { $yum::params::yum_redhat_pki_name:
	    ensure => present,
	    owner  => 'root',
	    group  => 'root',
	    mode   => '0644',
	    source => $yum::params::yum_redhat_pki_download,
	  }
	}
	class yum::config_repo{  
	  yumrepo { rhel-base: #创建yumrepo资源rhel-base
	    descr    => $yum::params::yum_redhat_descr, #设置描述信息
	    baseurl  => $yum::params::yum_redhat_baseurl, #设置yum源下载地址
	    enabled  => 1, #激活yum源
	    gpgcheck => 1, #设置要求通过pki校验
	    gpgkey   => $yum::params::yum_redhat_pki, #设置pki文件的下载位置
	    require  => Class['yum::config_key'], #要求这个文件必须存在
	    priority => 1, #设置repo的优先级为1(数字越小优先级越高)
	  }
	  yumrepo { rhel-puppet:
	    descr    => $yum::params::yum_puppet_descr,
	    baseurl  => $yum::params::yum_puppet_baseurl,
	    enabled  => 1,
	    gpgcheck => 0,
	    priority => 2,
	  }
	}

**5、创建init.pp文件**

由于params.pp文件中设置的变量名称引用太长，这里可以在init.pp中将变量名简化，方便引用。

	class yum(
	  $yum_redhat_descr        = $yum::params::yum_redhat_descr, #
	  $yum_puppet_descr        = $yum::params::yum_puppet_descr,
	  $yum_redhat_pki          = $yum::params::yum_redhat_pki,
	  $yum_redhat_baseurl      = $yum::params::yum_redhat_baseurl,
	  $yum_puppet_baseurl      = $yum::params::yum_puppet_baseurl,
	  $yum_redhat_pki_name     = $yum::params::yum_redhat_pki_name,
	  $yum_redhat_pki_download = $yum::params::yum_redhat_pki_download
	  ) inherits yum::params {   #设置这些变量依赖于yum::params类
	  include yum::config,yum::install  #包含所有子class
	}

因此、上面定义的class yum::config_key和yum::config_repo可以写成以下格式

	class yum::config_key{  #设置pki证书的一些属性及下载位置
	  file { $yum_redhat_pki_name:
	    ensure => present,
	    owner  => 'root',
	    group  => 'root',
	    mode   => '0644',
	    source => $yum_redhat_pki_download,
	  }
	}

	class yum::config_repo{  
	  yumrepo { rhel-base: #创建yumrepo资源rhel-base
	    descr    => $yum_redhat_descr, #设置描述信息
	    baseurl  => $yum_redhat_baseurl, #设置yum源下载地址
	    enabled  => 1, #激活yum源
	    gpgcheck => 1, #设置要求通过pki校验
	    gpgkey   => $yum_redhat_pki, #设置pki文件的下载位置
	    require  => Class['yum::config_key'], #要求这个文件必须存在
	    priority => 1, #设置repo的优先级为1(数字越小优先级越高)
	  }
	  yumrepo { rhel-puppet:
	    descr    => $yum_puppet_descr,
	    baseurl  => $yum_puppet_baseurl,
	    enabled  => 1,
	    gpgcheck => 0,
	    priority => 2,
	  }
	}

**6、创建puppet.conf和pki文件**

	[root@puppetmaster yum]# tree files
	files
	├── etc
	│   └── yum.conf  #可以从节点/etc/目录下copy一个yum.conf文件进行配置管理
	└── PM-GPG-KEY
	    ├── RPM-GPG-KEY-puppet-release  #自己做一个pki文件，如何做，请google
	    ├── RPM-GPG-KEY-redhat-release-rhel5  #在RHEL5系统/etc/pki/rpm-gpg/目录下面有对应的pki文件，将其命个别名即可
	    └── RPM-GPG-KEY-redhat-release-rhel6  #在RHEL6系统/etc/pki/rpm-gpg/目录下面有对应的pki文件，将其命个别名即可
	
	2 directories, 4 files

**7、应用到节点上**

	[root@puppetmaster modules]# vim /etc/puppet/manifests/site.pp 
	
	$puppetmaster = 'puppetmaster.kisspuppet.com'
	
	class environments{
	  include motd,puppet,yum
	}
	
	node default{
	   include environments
	}

**8、在agent1上进行测试**

	[root@agent1 yum.repos.d]# mv * /tmp/  #将所有的repo文件移动到/tmp目录下
	[root@agent1 yum.repos.d]# puppet agent -t #运行一次puppet更新动作，可以通过以下日志看出更新
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1395696487'
	info: create new repo rhel-puppet in file /etc/yum.repos.d/rhel-puppet.repo
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-puppet]/descr: descr changed '' to 'puppet rpm packages for rhel'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-puppet]/baseurl: baseurl changed '' to 'ftp://puppetmaster.kisspuppet.com/RHEL5U7'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-puppet]/enabled: enabled changed '' to '1'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-puppet]/gpgcheck: gpgcheck changed '' to '0'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-puppet]/priority: priority changed '' to '2'
	info: changing mode of /etc/yum.repos.d/rhel-puppet.repo from 600 to 644
	info: create new repo rhel-base in file /etc/yum.repos.d/rhel-base.repo
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-base]/descr: descr changed '' to 'rhel base rpm packages'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-base]/baseurl: baseurl changed '' to 'file:///media/cdrom/Server'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-base]/enabled: enabled changed '' to '1'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-base]/gpgcheck: gpgcheck changed '' to '1'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-base]/gpgkey: gpgkey changed '' to 'file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel5'
	notice: /Stage[main]/Yum::Config_repo/Yumrepo[rhel-base]/priority: priority changed '' to '1'
	info: changing mode of /etc/yum.repos.d/rhel-base.repo from 600 to 644
	notice: Finished catalog run in 0.51 seconds
	[root@agent1 yum.repos.d]# ls
	rhel-base.repo  rhel-puppet.repo
	[root@agent1 yum.repos.d]# cat rhel-base.repo  #查看更新的光盘源文件
	[rhel-base]
	name=rhel base rpm packages
	baseurl=file:///media/cdrom/Server
	enabled=1
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release-rhel5
	priority=1
	[root@agent1 yum.repos.d]# cat rhel-puppet.repo #插件更新的puppet源文件
	[rhel-puppet]
	name=puppet rpm packages for rhel
	baseurl=ftp://puppetmaster.kisspuppet.com/RHEL5U7
	enabled=1
	gpgcheck=0
	priority=2


**说明：**关于puppet的资源目前大概有48种，这里就不一一介绍了，详情可访问 [http://docs.puppetlabs.com/references/stable/type.html](http://docs.puppetlabs.com/references/stable/type.html)


