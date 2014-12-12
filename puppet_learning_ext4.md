#### Puppet扩展篇4-如何扩展master的SSL传输性能（nginx）


**描述：**puppet使用SSL(https)协议来进行通讯，默认情况下，puppet server端使用基于Ruby的WEBRick HTTP服务器。由于WEBRick HTTP服务器在处理agent端的性能方面并不是很强劲，因此需要扩展puppet，搭建nginx或者其他强劲的web服务器来处理客户的https请求。

**需要解决的问题：**

- 扩展传输方式：提高性能并增加Master和agent之间的并发连接数量。
- 扩展SSL：采用良好的SSL证书管理方法来加密Master和agent之间的通讯。
<!--more-->

Nginx+Passenger方式：

## 1、安装编译nginx所需要的开发包 ##

	[root@TKPUPT-M1 ~]# groupadd -g 3001 nginx
	[root@TKPUPT-M1 ~]# useradd -u 3001 -g 3001 nginx
	[root@TKPUPT-M1 ~]# yum install ruby-devel gcc make pcre-devel zlib-devel openssl-devel pam-devel curl-devel rpm-build

## 2、安装passenger（将gem软件包copy到本地） ##
备注：需要先将gem包下载到本地，当然也可以联网安装，会非常慢。

	[root@TKPUPT-M1 gem]# gem install --localhost rake rack passenger --no-rdoc --no-ri

## 3、解压nginx、pcre源码包 ##

	[root@TKPUPT-M1 gem]# tar xf pcre-8.32.tar.gz -C /usr/local/src/
	[root@TKPUPT-M1 gem]# tar xf nginx-1.4.2.tar.gz -C /usr/local/src/

## 4、编译并安装nginx ##
备注：主要是为了将模块passenger-config编译进来。

	[root@TKPUPT-M1 ~]# cd /usr/local/src/nginx-1.4.2/
	[root@TKPUPT-M1 nginx-1.4.2]# ./configure --user=nginx --group=nginx --prefix=/etc/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/src/pcre-8.32 --add-module=`passenger-config --root`/ext/nginx
	[root@TKPUPT-M1 nginx-1.4.2]# make && make install

## 5、与passenger结合 ##
备注：注意config.ru的属主和属组应该为puppet

	[root@TKPUPT-M1 nginx-1.4.2]# mkdir  -p /etc/puppet/rack/public
	[root@TKPUPT-M1 nginx-1.4.2]# cp /usr/share/puppet/ext/rack/config.ru  /etc/puppet/rack/
	[root@TKPUPT-M1 nginx-1.4.2]# chown -R puppet. /etc/puppet/rack/

## 6、复制启动脚本到 ##

	[root@TKPUPT-M1 init.d]# cp /root/gem/nginx /etc/init.d/
	[root@TKPUPT-M1 ~]# chmod a+x /etc/init.d/nginx 

## 7、配置nginx ##
备注：注意和puppet结合的证书名称及路径

	[root@TKPUPT-M1 gem]# vim /etc/nginx/conf/nginx.conf
	user  nginx nginx;
	worker_processes  1;
	pid        /var/run/nginx.pid;
	
	events {
	    worker_connections  1024;
	}
	
	http {
	    passenger_root /usr/lib/ruby/gems/1.8/gems/passenger-4.0.19;
	    passenger_ruby /usr/bin/ruby;
	
	    include       mime.types;
	    default_type  application/octet-stream;
	
	    sendfile        on;
	    keepalive_timeout  65;
	    
	    server {
	        listen 8140                ssl;
		server_name                puppetmaster;
		
		passenger_enabled          on;
		passenger_set_cgi_param    HTTP_X_CLIENT_DN $ssl_client_s_dn;
		passenger_set_cgi_param    HTTP_X_CLIENT_VERIFY $ssl_client_verify;
		proxy_buffer_size 4000k;
		proxy_buffering on;
		proxy_buffers 32 1280k;
		proxy_busy_buffers_size 17680k;
		client_max_body_size 10m;
		client_body_buffer_size 4096k;
		
		access_log /var/log/nginx/puppet_access.log;
		error_log /var/log/nginx/puppet_error.log;
		
		root /etc/puppet/rack/public;
		
		ssl off;
		ssl_session_timeout 5m;
		ssl_certificate /var/lib/puppet/ssl/certs/puppetmaster.pem;
		ssl_certificate_key /var/lib/puppet/ssl/private_keys/puppetmaster.pem;
		ssl_client_certificate /var/lib/puppet/ssl/certs/ca.pem;
		ssl_crl /var/lib/puppet/ssl/ca/ca_crl.pem;
		ssl_verify_client optional;
		
		ssl_ciphers SSLv2:-LOW:-EXPORT:RC4+RSA;
		ssl_prefer_server_ciphers on;
		ssl_verify_depth 1;
		ssl_session_cache shared:SSL:128m;
			
		# File sections
		location /production/file_content/files/ {
		  types { }
		  default_type application/x-raw;
		  alias /etc/puppet/files/;
	  }
	 }
	}
	
## 8、配置puppet.conf ##

	[root@TKPUPT-M1 ~]# vim /etc/puppet/puppet.conf 
	[master]
	    certname = puppetmaster
	    ca       = false
	    ssl_client_verify_header = HTTP_X_CLIENT_VERIFY
	    ssl_client_header = HTTP_X_CLIENT_DN

## 8、启动nginx ##

	[root@TKPUPT-M1 gem]# mkdir /var/log/nginx/
	[root@TKPUPT-M1 nginx-1.4.2]# /etc/init.d/puppetmaster stop
	[root@TKPUPT-M1 nginx-1.4.2]# chkconfig puppetmaster off
	[root@TKPUPT-M1 nginx-1.4.2]# /etc/init.d/nginx start
	[root@TKPUPT-M1 nginx-1.4.2]# chkconfig nginx on

## 9、测试 ##

在多个节点发起puppet agent -t命令动作，查看nginx日志看nginx+passenger是否代理成功。

	[root@TKPUPT-CA ~]# puppet  agent -t
	[root@TKPUPT-M1 ~]# tailf  /var/log/nginx/puppet_access.log


**参考：**[http://projects.puppetlabs.com/projects/1/wiki/Using_Passenger](http://projects.puppetlabs.com/projects/1/wiki/Using_Passenger)


