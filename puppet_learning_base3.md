 #### Puppet基础篇3-安装Puppet前期的准备工作


# 工欲善其事必先利其器 #

在安装Puppet之前是需要做很多预备工作的，比如网络地址规范、主机名、certname名、时间等等，也只有这些准备好了，才不至于在安装好puppet之后发现问题而后悔莫及。
**说明：**接下来的整套文档体系都是以本篇文档规范方案进行设计和扩充的，同样也是也是按照准生产的标准进行编写。
<!--more-->

## 一、网络地址规范 ##

	【HOSTNAME】                        【IP】				     【certname】		【operatingsystem】 
	puppetmaster.kisspuppet.com  192.168.100.110/24   puppetmaster_cert.kisspuppet.com	 RHEL6.4
	agent1.kisspuppet.com        192.168.100.111/24   agent1_cert.kisspuppet.com         RHEL5.7
	agent2.kisspuppet.com        192.168.100.112/24   agent2_cert.kisspuppet.com         RHEL5.8
	agent3.kisspuppet.com        192.168.100.123/24   agent3_cert.kisspuppet.com         RHEL6.4
	
**注：**192.168.100.*/24的网关为192.168.100.110   所有服务器的DNS1为192.168.100.110

**1、设置主机名**

	[root@puppetmaster ~]# vim /etc/sysconfig/network
	NETWORKING=yes
	HOSTNAME=puppetmaster.kisspuppet.com
	[root@agent1 ~]# vim /etc/sysconfig/network
	NETWORKING=yes
	NETWORKING_IPV6=no
	HOSTNAME=agent1.kisspuppet.com

**注：**agent2~agent3同上

**2、设置IP地址**

可通过`system-config-network`命令进行配置好后在进入配置文件进行修改

	[root@puppetmaster ~]# vim /etc/sysconfig/network-scripts/ifcfg-eth0 
	DEVICE=eth0
	TYPE=Ethernet
	ONBOOT=yes
	NM_CONTROLLED=yes
	BOOTPROTO=none
	IPADDR=192.168.100.110
	NETMASK=255.255.255.0
	GATEWAY=192.168.100.110
	DNS1=192.168.100.110
	IPV6INIT=no
	USERCTL=no

**注：**node1~node3同上

**3、关闭NetworkManager服务**

NetworkManager服务是RHEL图形界面管理网卡的服务，由于其开启会对网络造成影响，RHEL6默认是开启的，建议关闭。

	[root@puppetmaster ~]# /etc/init.d/NetworkManager stop
	Stopping NetworkManager daemon:                            [  OK  ]
	[root@puppetmaster ~]# chkconfig NetworkManager off

**注：**node1~node3同上

**4、关闭防火墙**

本实验主要是为了测试整个架构的功能，如果要测试防火墙，请另行解决。

	[root@puppetmaster ~]# /etc/init.d/iptables stop
	iptables: Flushing firewall rules:                         [  OK  ]
	iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
	iptables: Unloading modules:                               [  OK  ]
	[root@puppetmaster ~]# chkconfig iptables off

**注：**node1~node3同上

**5、关闭selinux**

	[root@puppetmaster ~]# sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config 

**注：**node1~node3同上

**6、设置key**
为了操作方便，设置公钥私钥，可通过puppetmaster端统一部署

	[root@puppetmaster ~]# ssh-keygen 
	Generating public/private rsa key pair.
	Enter file in which to save the key (/root/.ssh/id_rsa): 
	Enter passphrase (empty for no passphrase): 
	Enter same passphrase again: 
	Your identification has been saved in /root/.ssh/id_rsa.
	Your public key has been saved in /root/.ssh/id_rsa.pub.
	The key fingerprint is:
	ff:55:8d:31:34:b4:b3:6a:70:3b:aa:09:76:12:5b:8d root@puppetmaster.kisspuppet.com
	The key's randomart image is:
	+--[ RSA 2048]----+
	|             .+  |
	|             . o |
	|              =  |
	|         o     *.|
	|      . E o . o o|
	|       + . o o . |
	|      = . . = .  |
	|     . + . + o   |
	|        o.. .    |
	+-----------------+
	[root@puppetmaster ~]# for i in {1..3}; do ssh-copy-id -i 192.168.100.11$i; done
	The authenticity of host '192.168.100.111 (192.168.100.111)' can't be established.
	RSA key fingerprint is ae:db:c5:0c:0e:3f:8c:62:ea:a1:26:e2:09:63:18:32.
	Are you sure you want to continue connecting (yes/no)? yes
	Warning: Permanently added '192.168.100.111' (RSA) to the list of known hosts.
	root@192.168.100.111's password: 
	Now try logging into the machine, with "ssh '192.168.100.111'", and check in:
	
	  .ssh/authorized_keys
	
	to make sure we haven't added extra keys that you weren't expecting.
	...


**7、设置hosts文件**

puppet通信的前提是agent和master必须能够互相解析主机名。
当然，也可以设置DNS，在第四部分搭建kermit架构的时候会搭建DNS服务，现在先暂时通过hosts文件进行解析，可先设置好puppetmaster后，统一copy到所有节点上

	[root@puppetmaster ~]# vim /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	192.168.100.110 puppetmaster.kisspuppet.com  puppetmaster
	192.168.100.111 agent1.kisspuppet.com  agent1
	192.168.100.112 agent2.kisspuppet.com  agent2
	192.168.100.113 agent3.kisspuppet.com  agent3
	[root@puppetmaster ~]# for i in {1..3}; do scp /etc/hosts 192.168.100.11$i:/etc/; done
	hosts                                                                                  100%  354     0.4KB/s   00:00    
	hosts                                                                                  100%  354     0.4KB/s   00:00    
	hosts                                                                                  100%  354     0.4KB/s   00:00  

	[root@agent1 ~]# ping puppetmaster.kisspuppet.com #设置完成之后记得测试下
	PING puppetmaster.kisspuppet.com (192.168.100.110) 56(84) bytes of data.
	64 bytes from puppetmaster.kisspuppet.com (192.168.100.110): icmp_seq=1 ttl=64 time=0.327 ms
	64 bytes from puppetmaster.kisspuppet.com (192.168.100.110): icmp_seq=2 ttl=64 time=0.996 ms
	64 bytes from puppetmaster.kisspuppet.com (192.168.100.110): icmp_seq=3 ttl=64 time=1.00 ms
	
	--- puppetmaster.kisspuppet.com ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 1999ms
	rtt min/avg/max/mdev = 0.327/0.774/1.000/0.317 ms

**注意：**设置完网络后，可以通过类似**Xshell**这样的工具进行登录，方便操作

**二、配置本地光盘yum源**

由于我这边是vmware虚拟机操作，所以光盘是可以直接挂载到某一个目录里面，如果是物理机，建议将光盘里的文件全部copy到指定的一个目录里面，然后beaeurl指向它既可。

	[root@puppetmaster ~]# mkdir /media/cdrom
	[root@puppetmaster ~]# mount /dev/cdrom  /media/cdrom/
	mount: block device /dev/sr0 is write-protected, mounting read-only

	[root@puppetmaster ~]# cp /etc/yum.repos.d/rhel-source.repo /etc/yum.repos.d/rhel-base.repo 
	[root@puppetmaster ~]# vim /etc/yum.repos.d/rhel-base.repo 
	[rhel-base]
	name=Red Hat Enterprise Linux $releasever - $basearch - Source
	baseurl=file:///media/cdrom
	enabled=1
	gpgcheck=0
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

	[root@puppetmaster ~]# yum clean all
	Loaded plugins: product-id, refresh-packagekit, security, subscription-manager
	This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
	Cleaning repos: rhel-base
	Cleaning up Everything
	[root@puppetmaster ~]# yum install tree lrzsz   #测试
	Loaded plugins: product-id, refresh-packagekit, security, subscription-manager
	This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
	rhel-base                                                                                         | 3.9 kB     00:00 ... 
	rhel-base/primary_db                                                                              | 3.1 MB     00:01 ... 
	Setting up Install Process
	Resolving Dependencies
	...

**注：**node1~node3同上

**说明：**RHEL5的report在Server目录，所以在配置repo文件的时候参数 `baseurl=file:///media/cdrom/Server`

## 三、设置NTP服务器 ##

**1、配置NTP服务器**
设置ntp服务器和本地进行同步，当然如果联网也可以和外部服务器同步，这里只需要保证所有服务器时间一致。
原因是因为，puppetmaster和agent之间时间相差不得超过10分钟（好像是），而后期配置的mcollecitve服务端和客户端之间不能相差60秒

	[root@puppetmaster ~]# rpm -qa | grep ntp
	fontpackages-filesystem-1.41-1.1.el6.noarch
	ntpdate-4.2.4p8-3.el6.x86_64  #默认已经安装
	ntp-4.2.4p8-3.el6.x86_64 #默认已经安装
	[root@puppetmaster ~]# cp /etc/ntp.conf{,.bak}

	[root@puppetmaster ~]# vim /etc/ntp.conf
	driftfile /var/lib/ntp/drift
	logfile /var/log/ntp.log
	Broadcastdelay 0.008
	restrict default kod nomodify notrap nopeer noquery
	restrict -6 default kod nomodify notrap nopeer noquery
	restrict default ignore
	restrict 127.0.0.1
	restrict -6 ::1
	restrict 192.168.100.0  mask 255.255.255.0 notrap nomodify
	server  127.127.1.0     # local clock
	fudge   127.127.1.0 stratum 10   refid NIST
	includefile /etc/ntp/crypto/pw
	keys /etc/ntp/keys

	[root@puppetmaster ~]# /etc/init.d/ntpd start
	Starting ntpd:                                             [  OK  ]
	[root@puppetmaster ~]# chkconfig ntpd on


**2、节点测试并设置crontab**

	[root@agent1 ~]# ntpdate puppetmaster.kisspuppet.com
	 7 Mar 06:08:30 ntpdate[16411]: adjust time server 192.168.100.110 offset 0.049448 sec

	[root@agent1 ~]# crontab -l #可通过croutab -e命令设置
	*/30 * * * * /usr/sbin/ntpdate puppetmaster.kisspuppet.com >>/root/ntdate.log 2>&1 && /sbin/clock --systohc
	[root@agent1 ~]# /etc/init.d/crond reload
	Reloading cron daemon configuration:                       [  OK  ]


## 四、制作本地yum仓库 ##

本实验大部分包来自于[http://yum.puppetlabs.com](http://yum.puppetlabs.com)，部分包来自于EPEL和Gems官网，rabbitmq官方等，关于如何制作yum仓库，之前有文章写道[http://kisspuppet.com/2014/01/26/puppet_create_repo/](http://kisspuppet.com/2014/01/26/puppet_create_repo/)，这里在简单操作一下

	[root@puppetmaster RHEL6U4]# yum install createrepo #安装制作软件包的软件
	[root@puppetmaster RHEL6U4]# createrepo . #将本目录以及子目录下所有rpm包生产repodata
	Spawning worker 0 with 105 pkgs
	Workers Finished
	Gathering worker results
	
	Saving Primary metadata
	Saving file lists metadata
	Saving other metadata
	Generating sqlite DBs
	Sqlite DBs complete

**注：**RHEL5的repodata必须在RHEL5环境下运行createrpo命令生成

## 五、配置FTP服务器 ##

**1、安装并配置FTP服务器**

搭建FTP服务器的目的只要是为自定义yum仓库做准备

	[root@puppetmaster ~]# yum install vsftpd
	Loaded plugins: product-id, refresh-packagekit, security, subscription-manager
	This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
	rhel-base                                                                                         | 3.9 kB     00:00 ... 
	Setting up Install Process
	Resolving Dependencies
	--> Running transaction check
	---> Package vsftpd.x86_64 0:2.2.2-11.el6 will be installed
	--> Finished Dependency Resolution
	...

	[root@puppetmaster ~]# cp /etc/vsftpd/vsftpd.conf{,.bak}
	[root@puppetmaster ~]# vim /etc/vsftpd/vsftpd.conf
	anonymous_enable=YES
	local_enable=YES
	write_enable=YES
	local_umask=022
	anon_upload_enable=YES
	anon_root=/puppet   #匿名访问的目录
	anon_mkdir_write_enable=YES
	anon_other_write_enable=YES
	dirmessage_enable=YES
	xferlog_enable=YES
	connect_from_port_20=YES
	xferlog_file=/var/log/xferlog
	xferlog_std_format=YES
	listen=YES
	
	pam_service_name=vsftpd
	userlist_enable=YES
	tcp_wrappers=YES

	[root@puppetmaster ~]# /etc/init.d/vsftpd start
	Starting vsftpd for vsftpd:                                [  OK  ]
	[root@puppetmaster ~]# chkconfig vsftpd on

**2、在FTP共享目录里制作yum仓库**

将生成好的yum源copy到FTP共享目录中

	[root@puppetmaster ~]# ll /puppet/
	total 12
	drwxr-xr-x 4 root root 4096 Mar  7 06:21 RHEL5U7
	drwxr-xr-x 4 root root 4096 Mar  7 06:21 RHEL5U8
	drwxr-xr-x 6 root root 4096 Mar  7 06:21 RHEL6U4
	
	[root@puppetmaster ~]# ll /puppet/RHEL6U4/
	total 16600
	-rw-r--r-- 1 root root   87643 Mar  7 06:21 facter-1.7.3-1.el5.x86_64.rpm
	-rw-r--r-- 1 root root   87440 Mar  7 06:21 facter-1.7.3-1.el6.x86_64.rpm
	drwxr-xr-x 2 root root    4096 Mar  7 06:21 gem
	-rw-r--r-- 1 root root  634944 Mar  7 06:21 GeoIP-1.4.8-1.el6.x86_64.rpm
	-rw-r--r-- 1 root root  151654 Mar  7 06:21 keepalived-1.2.7-1.1.x86_64.rpm
	-rw-r--r-- 1 root root   10924 Mar  7 06:21 mcollective-2.2.4-1.el6.noarch.rpm
	-rw-r--r-- 1 root root   24596 Mar  7 06:21 mcollective-client-2.2.4-1.el6.noarch.rpm
	-rw-r--r-- 1 root root  759300 Mar  7 06:21 mcollective-common-2.2.4-1.el6.noarch.rpm
	drwxr-xr-x 3 root root    4096 Mar  7 06:21 mcollective-plugins
	drwxr-xr-x 2 root root    4096 Mar  7 06:21 mq
	-rw-r--r-- 1 root root  406588 Mar  7 06:21 nginx-1.0.15-5.el6.x86_64.rpm
	-rw-r--r-- 1 root root 1128352 Mar  7 06:21 puppet-2.7.23-1.el6.noarch.rpm
	-rw-r--r-- 1 root root 4509032 Mar  7 06:21 puppet-dashboard-1.2.23-1.el6.noarch.rpm
	-rw-r--r-- 1 root root   25596 Mar  7 06:21 puppet-server-2.7.23-1.el6.noarch.rpm
	-rw-r--r-- 1 root root 3729988 Mar  7 06:21 rabbitmq-server-3.1.5-1.el6.noarch.rpm
	drwxr-xr-x 2 root root    4096 Mar  7 06:21 repodata
	...

## 六、配置远程yum仓库 ##

	[root@puppetmaster ~]# vim /etc/yum.repos.d/rhel-puppet.repo
	[rhel-puppet]
	name=puppetlabs epel gems for rhel
	baseurl=ftp://puppetmaster.kisspuppet.com/RHEL6U4 #指向FTP服务器地址
	enabled=1
	gpgcheck=0
	
	[root@puppetmaster ~]# yum list | grep puppet-server #测试
	puppet-server.noarch                   2.7.25-1.el6                  rhel-puppet

**注：**node1~node3同上

## 七、重要软件版本选型 ##

目前puppet最成熟的版本为2.7.和3.3版本，两个版本都可以，本实验采用2.7版本。

	puppet-server 2.7.25-1 来自puppetlabs
	puppet 2.7.25-1 来自puppetlabs
	facter 1.7.5 来自puppetlabs
	puppet-dashboar 1.2.23 来自puppetlabs
	ruby 1.8.* 系统自带
	mcollective 2.2.4 来自puppetlabs
	activemq 5.5.0 来自puppetlabs
	rabbitmq-server 3.1.5 来自rabbitmq官网
	kermit-webui 1.2-1 来自kermit官网
	...



