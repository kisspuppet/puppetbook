#### Foreman架构的引入6-整合puppetmaster


**注：**以下内容是在**foreman1.6.3+puppet2.6.2**环境下进行操作。更多配置请参考官网[http://theforeman.org/manuals/1.6/index.html](http://theforeman.org/manuals/1.6/index.html)

安装好foreman和puppetmaster之后，接下来做的事情就是做整合，目前foreman可以管理puppet的环境、类、类里的变量、报告、facter等信息。接下来会逐一进行介绍。


# 1、首先要保证智能代理已经代理了puppet和puppet CA #

![Foreman安装](http://kisspuppet.com/img/foreman06-1.png)

代理puppet以及puppetCA，需要在foreman-proxy中开启。

	#配置代理puppet
	[root@puppetmaster162 ~]# cat /etc/foreman-proxy/settings.d/puppet.yml 
	---
	# Puppet management
	:enabled: true   #开启
	:puppet_conf: /etc/puppet/puppet.conf
	# valid providers:
	#   puppetrun   (for puppetrun/kick, deprecated in Puppet 3)
	#   mcollective (uses mco puppet)
	#   puppetssh   (run puppet over ssh)
	#   salt        (uses salt puppet.run)
	#   customrun   (calls a custom command with args)
	:puppet_provider: mcollective
	
	# customrun command details
	# Set :customrun_cmd to the full path of the script you want to run, instead of /bin/false
	:customrun_cmd: /bin/false
	# Set :customrun_args to any args you want to pass to your custom script. The hostname of the
	# system to run against will be appended after the custom commands.
	:customrun_args: -ay -f -s
	
	# whether to use sudo before the ssh command
	:puppetssh_sudo: false
	# the command which will be sent to the host
	:puppetssh_command: /usr/bin/puppet agent --onetime --no-usecacheonfailure
	# With which user should the proxy connect
	#:puppetssh_user: root
	#:puppetssh_keyfile: /etc/foreman-proxy/id_rsa
	
	# Which user to invoke sudo as to run puppet commands
	:puppet_user: root
	
	# URL of the puppet master itself for API requests
	:puppet_url: https://puppetmaster162.kisspuppet.com:8140
	# SSL certificates used to access the puppet master API
	:puppet_ssl_ca: /var/lib/puppet/ssl/certs/ca.pem
	:puppet_ssl_cert: /var/lib/puppet/ssl/certs/puppetmaster162.kisspuppet.com.pem
	:puppet_ssl_key: /var/lib/puppet/ssl/private_keys/puppetmaster162.kisspuppet.com.pem
	
	# Override use of Puppet's API to list environments, by default it will use only if
	# environmentpath is given in puppet.conf, else will look for environments in puppet.conf
	
	#:puppet_use_environment_api: true

	#配置代理puppet ca
	[root@puppetmaster162 ~]# cat /etc/foreman-proxy/settings.d/puppetca.yml 
	---
	# PuppetCA management
	:enabled: true
	:ssldir: /var/lib/puppet/ssl
	:puppetdir: /etc/puppet

# 2、管理puppet环境 #

## 2.1、配置puppetmaster环境 ##
puppet从2.6版本开始增加了“目录环境”的功能，更多详情请访问官网[https://docs.puppetlabs.com/puppet/latest/reference/environments.html](https://docs.puppetlabs.com/puppet/latest/reference/environments.html)

	[root@puppetmaster162 ~]# cat /etc/puppet/puppet.conf 
	[master]
		...
	    environmentpath  = /etc/puppet/environments
	    basemodulepath   = /etc/puppet/modules:/usr/share/puppet/modules
	    environment_timeout = 2  #多长时间刷新一次

	[root@puppetmaster162 ~]# ll /etc/puppet/environments/
	total 24
	drwxr-xr-x 4 root root 4096 Dec  5 16:46 development
	drwxr-xr-x 4 root root 4096 Dec  5 16:46 example42
	drwxr-xr-x 4 root root 4096 Dec  5 16:39 example_env
	drwxr-xr-x 5 root root 4096 Dec  5 17:03 production
	drwxr-xr-x 4 root root 4096 Dec  5 16:46 puppetlabs
	drwxr-xr-x 7 root root 4096 Dec  5 17:03 temp

**注意：**从以上配置可以看得出设置了两个环境。

## 2.2、foreman上导入puppet环境 ##

![Foreman安装](http://kisspuppet.com/img/foreman06-2.png)


# 3、管理puppet类 #

3.1、配置puppet类

注意以下几点：

- puppet.conf中basemodulepath的值所设置的路径为环境目录下所有环境的公共环境，里面的所有模块都会被其他环境搜索到（在没有配置environment.conf的前提下）

- 环境目录中每个环境目录里面默认应该包含manifests（存放主配置文件site.pp）目录和modules（存放模块）目录，目录结构如下。

		[root@puppetmaster162 environments]# tree production/
		production/
		├── environment.conf
		├── manifests
		│   └── site.pp
		├── modules
		│   ├── jenkins
		│   │   ├── files
		│   │   │   └── jenkins.repo
		│   │   ├── manifests
		│   │   │   ├── init.pp
		│   │   │   ├── install.pp
		│   │   │   ├── service.pp
		│   │   │   └── yum.pp
		│   │   ├── README
		│   │   └── templates
		│   └── motd
		│       ├── files
		│       │   └── motd
		│       ├── manifests
		│       │   └── init.pp
		│       └── templates
		└── system
		    └── ssh
		        ├── files
		        ├── manifests
		        │   ├── backup.pp
		        │   ├── config.pp
		        │   ├── init.pp
		        │   ├── install.pp
		        │   └── service.pp
		        ├── Modulefile
		        ├── README
		        ├── spec
		        │   └── spec_helper.rb
		        ├── templates
		        │   └── sshd_config.erb
		        └── tests
		            └── init.pp
		
		17 directories, 20 files

- 如果你想在一个环境里包含多个目录，每个目录里面又包含模块，应该添加environment.conf文件

		[root@puppetmaster162 environments]# ll temp/
		total 24
		-rw-r--r--  1 root root   95 Dec  5 17:03 environment.conf  #添加环境搜索配置文件
		drwxr-xr-x 11 root root 4096 Dec  5 17:02 juhailu
		drwxr-xr-x  2 root root 4096 Dec  5 16:48 kisspuppet
		drwxr-xr-x  4 root root 4096 Dec  5 16:56 lin
		drwxr-xr-x  2 root root 4096 Dec  5 16:48 manifests
		drwxr-xr-x  5 root root 4096 Dec  5 16:47 puppetlabs
		
		[root@puppetmaster162 environments]# ll temp/puppetlabs/
		total 12
		drwxr-xr-x 5 root root 4096 Dec  5 16:46 propuppet-demoapp
		drwxr-xr-x 5 root root 4096 Dec  5 16:46 puppetlabs-demoapp
		drwxr-xr-x 4 root root 4096 Dec  5 16:46 puppet-module-skeleton
	
		[root@puppetmaster162 environments]# cat temp/environment.conf #添加搜索路径
		modulepath = $basemodulepath:puppetlabs:modules:lin:modules:juhailu:modules:kisspuppet:modules

**注意：**添加搜索路径需要添加`$basemodulepath`，否则不会去搜索默认公共环境路径。

## 3.2、Foreman上导入puppet类 ##

![Foreman安装](http://kisspuppet.com/img/foreman06-3.png)


# 4、设置ENC #

## 4.1、通过节点直接管理模块 ##

![Foreman安装](http://kisspuppet.com/img/foreman06-4.png)

**备注：**添加主类就可以了

这样节点和模块就关联上了，相当于在site.pp中添加如下代码

node  puppetmaster162.kisspuppet.com{
   include ssh
}

## 4.2、通过组继承模块 ##

![Foreman安装](http://kisspuppet.com/img/foreman06-5.png)

![Foreman安装](http://kisspuppet.com/img/foreman06-6.png)

**备注：**如果使用组管理模块，不建议为某个节点单独勾选模块，否则你会发现如果先给节点添加了模块A，然后再给节点对应的组里添加了模块A，那么节点的puppet类哪里就会显示包含的类有两个同名的模块。

# 5、组与模块之间的管理 #

## 5.1、添加配置组 ##

**注：**foreman从1.5版本开始增加了“配置组”功能，可以将多个模块添加到“配置组”，然后给配置组命名，这样，主机组在勾选模块的时候，只需要勾选配置组即可集成里面所有的模块

![Foreman安装](http://kisspuppet.com/img/foreman06-7.png)
![Foreman安装](http://kisspuppet.com/img/foreman06-8.png)

# 6、查看设置是否成功 #

![Foreman安装](http://kisspuppet.com/img/foreman06-9.png)

![Foreman安装](http://kisspuppet.com/img/foreman06-10.png)

	#可以通过以下方式查看，前提是需要先运行node.rb，可通过"puppet agent"命令或者"node.rb  <certname>" 进行触发。
	[root@puppetmaster162 ~]# cat /var/lib/puppet/yaml/foreman/puppetmaster162.kisspuppet.com.yaml 
	---
	classes:
	  ssh: 
	parameters:
	  puppetmaster: puppetmaster162.kisspuppet.com
	  hostgroup: prd
	  root_pw: 
	  foreman_env: production
	  owner_name: Admin User
	  owner_email: root@kisspuppet.com

设置以上信息，可以完成ENC的功能，基本可以保障节点和class之间的勾连。可以在节点通过puppet agent命令进行测试。至于如何在foreman上进行推送，关注后续文章。