#### Puppet扩展篇5-通过多进程增强master的负载均衡能力（nginx+mongrel）



当puppetmaster管理的主机越来越多时，puppetmaster本身性能会存在性能瓶颈问题，除了增加服务器扩充puppetmaster的数量增加puppetmaster整体性能外，也可以通过单台扩充puppetmaster的进程数来增加puppetmaster的性能。

以下是通过nginx+mongrel负载均衡puppetmaster的进程，由nginx向所有puppetagent提供认证服务，除此之外的其他puppetmaster功能的实现由nginx转向puppetmaster其中一个进程去处理即可。而nginx的upstream字段里面所包含的地址填写为127.0.0.1指向puppetmaster进程，提高了安全性。
<!--more-->

**备注：**nginx+mongrel只支持puppet2.7之前版本（包括2.7版本在内）。

## 1、安装相关软件包 ##

	[root@puppetserver yum.repos.d]# yum install rubygem-mongrel nginx

## 2、增加puppet端口 ##

	[root@puppetserver yum.repos.d]# vim /etc/sysconfig/puppetmaster
	PUPPETMASTER_PORTS=( 18140 18141 18142 18143 )
	PUPPETMASTER_EXTRA_OPTS="--servertype=mongrel --ssl_client_header=HTTP_X_SSL_SUBJECT"
## 3、配置nginx服务 ##
添加upstream字段，注意ssl认证证书的路径

	[root@puppetserver nginx]# vim nginx.conf
	user              nginx nginx;
	worker_processes  4;
	error_log  /var/log/puppet/nginx-puppet.log notice;
	pid        /var/run/nginx.pid;
	events {
	    worker_connections  1024;
	}
	http {
	    default_type        application/octet-stream;
	    sendfile        on;
	    tcp_nopush     on;
	    keepalive_timeout  65;
	    tcp_nodelay         on;
	    large_client_header_buffers 16      4k;
	    proxy_buffers                       128     4k;
	    upstream puppetmaster {
	        server 127.0.0.1:18140;
	        server 127.0.0.1:18141;
	        server 127.0.0.1:18142;
	        server 127.0.0.1:18143;
	    }
	    server {
	        listen 8140;
	        root    /etc/puppet;
	        ssl                     on;
	        ssl_session_timeout     5m;
	        ssl_certificate         /var/lib/puppet/ssl/certs/puppetserver.kisspuppet.com.pem;
	        ssl_certificate_key     /var/lib/puppet/ssl/private_keys/puppetserver.kisspuppet.com.pem;
	        ssl_client_certificate  /var/lib/puppet/ssl/ca/ca_crt.pem;
	        ssl_crl                 /var/lib/puppet/ssl/ca/ca_crl.pem;
	        ssl_ciphers             SSLv2:-LOW:-EXPORT:RC4+RSA;
	        ssl_verify_client       optional;
	        location / {
	            proxy_pass          http://puppetmaster;
	            proxy_redirect      off;
	            proxy_set_header    Host            $host;
	            proxy_set_header    X-Real-IP       $remote_addr;
	            proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
	            proxy_set_header    X-Client-Verify $ssl_client_verify;
	            proxy_set_header    X-Client-DN     $ssl_client_s_dn;
	            proxy_set_header    X-SSL-Subject   $ssl_client_s_dn;
	            proxy_set_header    X-SSL-Issuer    $ssl_client_i_dn;
	            proxy_read_timeout  65;
	        }
	    }
	}

## 4、分别启动nginx服务和puppetmaster服务 ##

	[root@puppetserver1poc ~]# /etc/rc.d/init.d/nginx restart
	Stopping nginx:                                            [FAILED]
	Starting nginx:                                            [  OK  ]
	[root@puppetserver1poc ~]# /etc/rc.d/init.d/puppetmaster start
	Starting puppetmaster: 
	Port: 18140                                                [  OK  ]
	Port: 18141                                                [  OK  ]
	Port: 18142                                                [  OK  ]
	Port: 18143                                                [  OK  ]

## 5、查看监听端口 ##

	[root@puppetserver1poc ~]# netstat -nlp | grep 814
	tcp        0      0 0.0.0.0:8140                0.0.0.0:*                   LISTEN      6224/nginx          
	tcp        0      0 127.0.0.1:18140             0.0.0.0:*                   LISTEN      6271/ruby           
	tcp        0      0 127.0.0.1:18141             0.0.0.0:*                   LISTEN      6312/ruby           
	tcp        0      0 127.0.0.1:18142             0.0.0.0:*                   LISTEN      6351/ruby           
	tcp        0      0 127.0.0.1:18143             0.0.0.0:*                   LISTEN      6390/ruby   
	3.5.6	通过进程查看运行状况
	[root@puppetserver1poc ~]# ps -ef | grep ruby
	puppet    5422     1  1 13:58 ?        00:00:22 /usr/bin/ruby /usr/sbin/puppetmasterd
	root      6431     1  0 14:10 ?        00:00:01 ruby /usr/sbin/mcollectived --pid=/var/run/mcollectived.pid --config=/etc/mcollective/server.cfg
	puppet    7139     1  0 14:25 ?        00:00:00 /usr/bin/ruby /usr/sbin/puppetmasterd --servertype=mongrel --servertype=mongrel --ssl_client_header=HTTP_X_SSL_SUBJECT --masterport=18140 --pidfile=/var/run/puppet/puppetmaster.18140.pid
	puppet    7171     1  0 14:25 ?        00:00:00 /usr/bin/ruby /usr/sbin/puppetmasterd --servertype=mongrel --servertype=mongrel --ssl_client_header=HTTP_X_SSL_SUBJECT --masterport=18141 --pidfile=/var/run/puppet/puppetmaster.18141.pid
	puppet    7203     1  0 14:25 ?        00:00:00 /usr/bin/ruby /usr/sbin/puppetmasterd --servertype=mongrel --servertype=mongrel --ssl_client_header=HTTP_X_SSL_SUBJECT --masterport=18142 --pidfile=/var/run/puppet/puppetmaster.18142.pid
	puppet    7235     1  0 14:25 ?        00:00:00 /usr/bin/ruby /usr/sbin/puppetmasterd --servertype=mongrel --servertype=mongrel --ssl_client_header=HTTP_X_SSL_SUBJECT --masterport=18143 --pidfile=/var/run/puppet/puppetmaster.18143.pid
	root      7243  3858  0 14:26 pts/3    00:00:00 grep ruby

## 6、通过日志查看运行状况 ##

	[root@puppetserver1poc nodes]# tailf  /var/log/nginx/access.log 
	192.168.100.127 - - [25/Nov/2013:16:42:49 +0800] "POST /production/catalog/agent2.kisspuppet.com HTTP/1.1" 200 570 "-" "-"
	192.168.100.127 - - [25/Nov/2013:16:42:52 +0800] "PUT /production/report/agent2.kisspuppet.com HTTP/1.1" 200 58 "-" "-"
	192.168.100.126 - - [25/Nov/2013:16:42:54 +0800] "GET /production/file_metadatas/plugins?links=manage&checksum_type=md5&&ignore=---+%0A++-+%22.svn%22%0A++-+CVS%0A++-+%22.git%22&recurse=true HTTP/1.1" 404 56 "-" "-"
	192.168.100.126 - - [25/Nov/2013:16:42:54 +0800] "GET /production/file_metadata/plugins? HTTP/1.1" 404 36 "-" "-"
	192.168.100.126 - - [25/Nov/2013:16:42:55 +0800] "POST /production/catalog/agent1.kisspuppet.com HTTP/1.1" 200 570 "-" "-"
	192.168.100.126 - - [25/Nov/2013:16:42:58 +0800] "PUT /production/report/agent1.kisspuppet.com HTTP/1.1" 200 58 "-" "-"
	192.168.100.125 - - [25/Nov/2013:16:43:07 +0800] "GET



