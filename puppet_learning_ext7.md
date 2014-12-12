#### Puppet扩展篇7-puppet代码与版本控制系统的结合



# 一、介绍 #

通过安装部署Puppet C/S模型，实现Puppet Server端管理所有被控制机的整个生命周期：从初始化到软件升级、从配置文件创建到测试部署、从系统维护到服务器迁移等。Puppet能够持续化的与被控制机进行交互，从而实现配置文件的及时检测更新。结合SVN版本控制系统，puppet可在更新之前将当前正在运行的环境以版本的方式保存到SVN版本控制系统中，方便以后通过puppet更新出错或者需要回滚到之前的某一个环境时快速恢复。

<!--more-->

# 二、环境介绍 #

	序号	服务器类型	        		 版本/IP参数
	 1	PuppetMaster		 RHEL6.4 x86_64（192.168.100.110）
	 2	PuppetAgent			 RHEL5.8 x86_64（192.168.100.111）和RHEL5.7 x86_64（192.168.100.112）
	 3	SVN Service端	 RHEL6.4 x86_64（192.168.100.110）
	 4	SVN Service端	 RHEL6.4 x86_64（192.168.100.110）和Windows 8.1 x86_64(192.168.100.2)

	编号	  类型	   主机名/软件名称	系统/软件版本					其他信息
	 1	Software	Subversion		1.6.11-7	rpm 				package
	 2	Software	TortoiseSVN		1.8.2.24708-x64-svn-1.8.3		msi

# 三、部署流程 #

## 1 SVN Server端部署 ##

**1.1 安装相关软件包**

	[root@puppetserver ~]# yum install subversion
	[root@puppetserver ~]# svnserve –version  #通过查看版本验证安装是否成功
	svnserve, version 1.6.11 (r934486)
	   compiled Apr 12 2012, 11:09:11
	
	Copyright (C) 2000-2009 CollabNet.
	Subversion is open source software, see http://subversion.tigris.org/
	This product includes software developed by CollabNet (http://www.Collab.Net/).
	
	The following repository back-end (FS) modules are available:
	
	* fs_base : Module for working with a Berkeley DB repository.
	* fs_fs : Module for working with a plain file (FSFS) repository.
	
	Cyrus SASL authentication is available.


**1.2 创建第一个版本库**

	[root@puppetserver ~]# mkdir /svndata
	[root@puppetserver ~]# svnadmin create /svndata/puppet
	[root@puppetserver ~]# ll /svndata/puppet/
	total 24
	drwxr-xr-x 2 root root 4096 Oct 22 13:29 conf
	drwxr-sr-x 6 root root 4096 Oct 22 13:29 db
	-r--r--r-- 1 root root    2 Oct 22 13:29 format
	drwxr-xr-x 2 root root 4096 Oct 22 13:29 hooks
	drwxr-xr-x 2 root root 4096 Oct 22 13:29 locks
	-rw-r--r-- 1 root root  229 Oct 22 13:29 README.txt

## 2 通过Apache+ssl安全认证访问SVN服务器 ##

**2.1	安装相关软件包**

	[root@puppetserver ~]# yum install httpd httpd-devel mod_dav_svn

**2.2	创建SVN虚拟主机**

	[root@puppetserver svndata]# vim /etc/httpd/conf.d/subversion.conf
	LoadModule dav_svn_module     modules/mod_dav_svn.so
	LoadModule authz_svn_module   modules/mod_authz_svn.so
	Listen 8142
	<VirtualHost *:8142>
	<Location /svndata>
	DAV svn
	SVNListParentPath on
	SVNPath "/svndata/puppet"
	AuthType Basic
	AuthName "Subversion repository"
	AuthUserFile "/svndata/puppet/conf/authfile"
	#AuthzSVNAccessFile /svndata/puppet/conf/svn-acl-conf
	Require valid-user
	SVNAutoversioning on
	ModMimeUsePathInfo on
	</Location>
	</VirtualHost>

**2.3	创建svn权限配置文件**

	[root@puppetserver svndata]# vim puppet/conf/authz  
	[groups]
	admin = puppet
	[admin:/]
	@admin = rw
	[/]
	* = r
	[$name:/]
	test = rw">>/svndata/puppet/conf/authz

	2.4	创建用户名及密码并设置相应权限
	[root@puppetserver ~]# /usr/bin/htpasswd -c /svndata/puppet/conf/authfile puppet #创建SVN服务器账户puppet密码为redhat
	New password: redhat
	Re-type new password: redhat
	Adding password for user puppet
	[root@puppetserver ~]# chown apache /svndata/puppet -R
	[root@puppetserver ~]# echo "puppet = redhat" >>/svndata/puppet/conf/passwd

**2.5	配置SVN服务信息**

	[root@puppetserver svndata]# vim /svndata/puppet/conf/svnserve.conf 
	[general] 
	anon-access = none
	auth-access = write
	password-db = /svndata/puppet/conf/passwd
	authz-db = /svndata/puppet/conf/authz
	realm = puppet Repository

**2.6	通过浏览器测试访问**

	[root@puppetserver svndata]# /etc/rc.d/init.d/httpd restart #重启httpd服务
	http://192.168.100.110:8142/svndata/

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-1.png)

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-2.png)

**2.7	通过其他linux节点访问测试**

	[root@agent1 ~]# svn checkout http://192.168.100.110:8142/svndata/  /mnt/
	Authentication realm: <http://192.168.100.110:8142> Puppet Subversion repository
	Password for 'root': 
	Authentication realm: <http://192.168.100.110:8142> Puppet Subversion repository
	Username: puppet 
	Password for 'puppet':
	-----------------------------------------------------------------------
	ATTENTION!  Your password for authentication realm:
	
	   <http://192.168.100.110:8142> Puppet Subversion repository
	
	can only be stored to disk unencrypted!  You are advised to configure
	your system so that Subversion can store passwords encrypted, if
	possible.  See the documentation for details.
	
	You can avoid future appearances of this warning by setting the value
	of the 'store-plaintext-passwords' option to either 'yes' or 'no' in
	'/root/.subversion/servers'.
	-----------------------------------------------------------------------
	Store password unencrypted (yes/no)? no
	Checked out revision 0.

**2.8	通过Windows客户端TortoiseSVN访问测试**

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-3.png)

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-4.png)

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-5.png)

**备注：**由于还为import版本，所以查看的内容为空

## 3 整合puppet server端 ##

**3.1	将puppet server模块目录导入到版本库中**

	[root@puppetserver ~]# svn import /etc/puppet/environments/testing
	http://192.168.100.110:8142/svndata/puppet -m "Puppet Initial repository" 
	Authentication realm: <http://192.168.100.110:8142> Puppet Subversion repository
	Password for 'root': 
	Authentication realm: <http://192.168.100.110:8142> Puppet Subversion repository
	Username: puppet
	Password for 'puppet': 
	Adding         /etc/puppet/environments/testing/groups
	Adding         /etc/puppet/environments/testing/groups/modules
	Adding         /etc/puppet/environments/testing/groups/modules/grub
	Adding         /etc/puppet/environments/testing/groups/modules/grub/files
	Adding         /etc/puppet/environments/testing/groups/modules/grub/manifests
	…
	Committed revision 1.

**备注：**由于SVN服务器端和puppetserver在同一台服务器上，也可以通过以下方式进行导入

	[root@puppetserver ~]# svn import /etc/puppet/environments/testing  
	file:///svndata/puppet -m "Puppet Initial repository"

**3.2	通过IE浏览器访问SVN服务器**

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-6.png)

**3.3	通过Windows客户端TortoiseSVN checkout最新的版本库到本地**

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-7.png)

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-8.png)

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-9.png)

**3.4	删除puppetserver端testing目录，并将版本库中的数据导出**

	[root@puppetserver ~]# cd /etc/puppet/environments/testing/
	[root@puppetserver testing]# rm -rf *  #删除之前建议备份
	[root@puppetserver testing]# svn checkout
	http://192.168.100.110:8142/svndata/puppet  /etc/puppet/environments/testing 
	Authentication realm: <http://192.168.100.110:8142> Puppet Subversion repository
	Password for 'puppet':
	Please type 'yes' or 'no': no
	A    groups
	A    groups/modules
	A    groups/modules/grub
	A    groups/modules/grub/files
	A    groups/modules/grub/manifests
	Checked out revision 1.
	[root@puppetserver testing]# ls -a
	.  ..  agents  environment  groups  manifests  .svn
	[root@puppetserver testing]# ls .svn/  #每个目录下面都会生成.svn隐藏目录，用于保存当前版本的信息
	all-wcprops  entries  prop-base  props  text-base  tmp
	备注：checkout之后，在/etc/puppet/environments/testing目录下就会有一份SVN服务器上最新版本的副本。

## 4 部署SVN hooks ##

**4.1	设置pre-commit**

设置pre-commit钩子可以提交文件到SNV服务器之前对puppet语法进行检查，语法通过则提交成功，语法错误则提交失败。

	[root@puppetserver hooks]# chmod 774 pre-commit^C
	[root@puppetserver hooks]# cp pre-commit.tmpl pre-commit
	[root@puppetserver hooks]# chmod 774 pre-commit 
	[root@puppetserver hooks]# vim pre-commit 
	#!/bin/sh
	# SVN pre-commit hook to check Puppet syntax for .pp files
	# Modified from http://mail.madstop.com/pipermail/puppet-users/2007-March/002034.html
	# Access http://projects.puppetlabs.com/projects/1/wiki/puppet_version_control
	REPOS="$1"
	TXN="$2"
	tmpfile=`mktemp`
	export HOME=/
	SVNLOOK=/usr/bin/svnlook
	$SVNLOOK changed -t "$TXN" "$REPOS" | awk '/^[^D].*\.pp$/ {print $2}' | while read line
	do
	    $SVNLOOK cat -t "$TXN" "$REPOS" "$line" > $tmpfile
	    if [ $? -ne 0 ]
	    then
	        echo "Warning: Failed to checkout $line" >&2
	    fi
	#    puppet --color=false --confdir=/etc/puppet --vardir=/var/lib/puppet --parseonly --ignoreimport $tmpfile >>/var/log/puppet/svn_pre-commit.log 2>&1
	    puppet --color=false --confdir=/etc/puppet --vardir=/var/lib/puppet --parser --ignoreimport $tmpfile >>/var/log/puppet/svn_pre-commit.log 2>&1
	    if [ $? -ne 0 ]
	    then
	        echo "Puppet syntax error in $line." >>/var/log/puppet/svn_pre-commit.log 2>&1
	        exit 2
	    fi
	done
	res=$?
	rm -f $tmpfile
	if [ $res -ne 0 ]
	then
	    exit $res
	fi

**4.2	设置post-commit**

设置post-commit钩子可以在正确提交文件至SVN服务器之后，puppetmaster的模块目录`/etc/puppet/environments/testing`会自动从SNV服务器上update最新的版本库到本地。

	#!/bin/sh
	# POST-COMMIT HOOK
	REPOS="$1"
	REV="$2"
	#mailer.py commit "$REPOS" "$REV" /path/to/mailer.conf
	export LANG=en_US.UTF-8
	SVN=/usr/bin/svn
	PUPPET_DIR=/etc/puppet
	#/usr/bin/svn  up /etc/puppet -non-interactive
	$SVN  update $PUPPET_DIR --username puppet --password 123.com >>/var/log/puppet/svn_post-commit.log

## 5 SVN Client端部署测试 ##

**5.1	本地测试**

1）导出版本数据库文件到本地

	[root@puppetserver ~]# svn checkout file:///svndata/puppet  /puppet/puppet
 
2）、创建并添加新的目录及文件

	[root@puppetserver puppet]# svn add ssh

3）、将修改后的文件提交到SVN服务器，此时版本库版本加1

	[root@puppetserver .svn]# svn commit -m "add ssh modules" /puppet/puppet/*

**5.2	远程测试（Linux）**

	[root@agent1 svndata]#  svn checkout http://172.16.200.100/svndata/  /mnt/

**5.3	客户端TortoiseSVN测试（Windows）**

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-12.png)

![svn版本控制测试界面](http://kisspuppet.com/img/svn-puppet-13.png)


