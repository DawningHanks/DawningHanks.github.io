# Steps on install LNMP on CentOS
[Step 1]前期准备
1.1 关闭防火墙于selinux
iptables -F #关闭防火墙
setenforce 0  #关闭selinux
getenforce    #查看selinux状态
vim /etc/selinux/config  #在配置文件中修改selinux
[root@localhost html]# vim /etc/selinux/config 


# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled   #改成disabled 
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

:wq


1.2 下载源码包
wget install yum
mysql-5.6.34.tar.gz
  wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.6.34.tar.gz
nginx-1.12.1.tar.gz
  wget http://nginx.org/download/nginx-1.12.2.tar.gz
php-5.6.30.tar.gz
  wget https://www.php.net/distributions/php-5.6.30.tar.gz
wordpress5.3.2.tar.gz
  wget https://wordpress.org/latest.tar.gz
  
1.3 开发环境与编译工具
yum -y groupinstall Development Tools
yum -y install make gcc-c++ cmake bison-devel ncurses-devel libaio libaio-devel perl-Data-Dumper net-tools 
yum -y install zlib zlib-devel openssl openssl--devel pcre pcre-devel libxml2*

[Step 2]安装MySQL
2.1 查看并删除系统的mysql服务
rpm -qa | grep mysql  #如果查询到mysql,就是用rpm -e命令来卸载了原来的mysql
rm -rf /etc/my.cnf    #删除原来mysql的配置文件

2.2 解压并复制文件
tar -zxvf mysql-5.6.35-linux-glibc2.5-x86_64.tar.gz  #在下载目录中解压文件
mkdir /usr/local/mysql   #创建安装目录,方便以后管理
cp  -R ./mysql-5.6.35-linux-glibc2.5-x86_64/* /usr/local/mysql/  #复制解压文件的目录中

2.3 切换目录并安装
#使用cmake工具进行安装mysql
cd /usr/local/mysql/
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/data/mysql -DSYSCONFDIR=/etc -DWITH_MYISAM_STORAGE_ENGINE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_MEMORY_STORAGE_ENGINE=1 -DWITH_READLINE=1 -DMYSQL_UNIX_ADDR=/var/lib/mysql/mysql.sock -DMYSQL_TCP_PORT=3306 -DENABLED_LOCAL_INFILE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DEXTRA_CHARSETS=all -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci
make install

2.4 创建数据目录,执行用户并执行初始化脚本
#初始化脚本
 mkdir /data
 useradd -s /sbin/nologin mysql
 chmod +x ./scripts/mysql_install_db 
 ./scripts/mysql_install_db --user=mysql --datadir=/data/mysql  

2.5 复制配置文件并配置
 #配置文件及启动脚本
cp support-files/my-default.cnf /etc/my.cnf 
cp support-files/mysql.server /etc/init.d/mysql.server  
chmod 755 /etc/init.d/mysql.server 

在`vim /etc/my.cnf`配置如下:
[root@localhost mysql]# vim /etc/my.cnf

# *** default location during install, and will be replaced if you
# *** upgrade to a newer version of MySQL.

[mysqld]

# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M

# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin

# These are commonly set, remove the # and set as required.
# basedir = .....
# datadir = .....
datadir =/data/mysql   #添加这个数据存放路径
# port = .....
# server_id = .....
# socket = .....
socket = /var/lib/mysql/mysql.sock    #还有这个,其他暂时不用动
...省略...

:wq

2.6 启动mysql并初始设置
/etc/init.d/mysql.server start   #启动mysql
/usr/local/mysql/bin/mysql_secure_installation  #初始设置,包含root密码设置等等

2.7 检查服务是否启动
[root@localhost html]# netstat -lntp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1265/master         
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1052/sshd           
tcp6       0      0 ::1:25                  :::*                    LISTEN      1265/master         
tcp6       0      0 :::3306                 :::*                    LISTEN      21121/mysqld    #3306已经启动监听     
tcp6       0      0 :::22                   :::*                    LISTEN      1052/sshd  

————————————————————————————————————————MySQL安装完成——————————————————————————————————————————————

[Step 3]安装PHP并配置
3.1 解压并进入目录
cd /usr/local/src/  #进入下载目录
tar -zxvf php-5.6.30.tar.gz 
cd php-5.6.30/  #进入解压目录

3.2 添加用户并编译安装
#添加php-fpm用户
 useradd -s /sbin/nologin php-fpm  
#可以通过./configure --help来按需要进行添加编译选项
 ./configure --prefix=/usr/local/php-fpm --with-config-file-path=/usr/local/php-fpm/etc --enable-fpm --with-fpm-user=php-fpm --with-fpm-group=php-fpm --with-mysql=/usr/local/mysql --with-mysqli=/usr/local/mysql/bin/mysql_config --with-pdo-mysql=/usr/local/mysql --with-mysql-sock=/tmp/mysql.sock 
 make && make install  

3.3 复制配置文件并设置权限
#复制配置文件及启动脚本
cp php.ini-production /usr/local/php-fpm/etc/php.ini
cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm 
chmod 755 /etc/init.d/php-fpm

3.4 创建配置文件
创建vim /usr/local/php-fpm/etc/php-fpm.conf,内容如下:

[root@localhost php-5.6.30]# vim /usr/local/php-fpm/etc/php-fpm.conf

[global]
pid = /usr/local/php-fpm/var/run/php-fpm.pid
error_log = /usr/local/php-fpm/var/log/php-fpm.log
[www]
listen = /tmp/php-fcgi.sock   #监听ip和端口，端口默认为9000
listen.mode = 666    #用来定义php-fcgi.sock文件的权限
user = php-fpm  
group = php-fpm
pm = dynamic   #后面这些都是关于进程的信息
pm.max_children = 50
pm.start_servers = 20
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 500
rlimit_files = 1024
               
:wq

Note: 各行注释仅用于说明，不能写入配置文件中，否则会报错！

3.5 检查配置文件并启动
/usr/local/php-fpm/sbin/php-fpm -t 
chkconfig --add php-fpm 
chkconfig php-fpm on  
service php-fpm start

3.6 检查服务是否启动
[root@localhost src]# ps aux | grep php
root     15636  0.0  0.0  55784  3568 ?        Ss   04:27   0:00 php-fpm: master process (/usr/local/php-fpm/etc/php-fpm.conf)
php-fpm  15637  0.0  0.0  55784  3296 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15638  0.0  0.0  55784  3296 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15639  0.0  0.0  55784  3296 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15640  0.0  0.0  55784  3296 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15641  0.0  0.0  55784  3300 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15642  0.0  0.0  55784  3300 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15643  0.0  0.0  55784  3300 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15644  0.0  0.0  55784  3300 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15645  0.0  0.0  55784  3300 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15646  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15647  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15648  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15649  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15650  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15651  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15652  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15653  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15654  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15655  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
php-fpm  15656  0.0  0.0  55784  3304 ?        S    04:27   0:00 php-fpm: pool www
root     17533  0.0  0.0 112732   968 pts/4    S+   04:29   0:00 grep --color=auto php

这里看到php已经启动,说明安装成功了。

————————————————————————————————————————PHP安装完成——————————————————————————————————————————————

[Step 4]安装并配置nginx
4.1 解压并进入目录
cd /usr/local/src/  #进入下载目录
tar -zxvf nginx-1.12.1.tar.gz 
cd nginx-1.12.1

4.2 编译安装
#可以使用./configure --help查看更多信息
./configure --prefix=/usr/local/nginx 
make
make install

4.3 创建启动脚本并修改权限
chkconfig这个和下面的选项好正确,否则会报service nginx does not support chkconfig的错误,
vim /etc/init.d/nginx

#!/bin/bash
# chkconfig: - 30 21   
# description: http service.
# Source Function Library
. /etc/init.d/functions
# Nginx Settings
NGINX_SBIN="/usr/local/nginx/sbin/nginx"
NGINX_CONF="/usr/local/nginx/conf/nginx.conf"
NGINX_PID="/usr/local/nginx/logs/nginx.pid"
RETVAL=0
prog="Nginx"
start() 
{
    echo -n $"Starting $prog: "
    mkdir -p /dev/shm/nginx_temp
    daemon $NGINX_SBIN -c $NGINX_CONF
    RETVAL=$?
    echo
    return $RETVAL
}
stop() 
{
    echo -n $"Stopping $prog: "
    killproc -p $NGINX_PID $NGINX_SBIN -TERM
    rm -rf /dev/shm/nginx_temp
    RETVAL=$?
    echo
    return $RETVAL
}
reload()
{
    echo -n $"Reloading $prog: "
    killproc -p $NGINX_PID $NGINX_SBIN -HUP
    RETVAL=$?
    echo
    return $RETVAL
}
restart()
{
    stop
    start
}
configtest()
{
    $NGINX_SBIN -c $NGINX_CONF -t
    return 0
}
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  reload)
        reload
        ;;
  restart)
        restart
        ;;
  configtest)
        configtest
        ;;
  *)
        echo $"Usage: $0 {start|stop|reload|restart|configtest}"
        RETVAL=1
esac
exit $RETVAL

esc + :wq 保存并退出。


chmod 755 /etc/init.d/nginx

4.4 加入服务列表并开机启动
chkconfig --add nginx
chkconfig nginx on

4.5 修改nginx.conf文件
在/usr/local/nginx/conf/目录下修改nginx.conf文件,为了保证安全先备份原来conf文件并创建新的配置文件.
mv /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf.bak

配置新的vim /usr/local/nginx/conf/nginx.conf文件,内容如下:
user nobody nobody;
worker_processes 2;
error_log /usr/local/nginx/logs/nginx_error.log crit;
pid /usr/local/nginx/logs/nginx.pid;
worker_rlimit_nofile 51200;
events
{
    use epoll;
    worker_connections 6000;
}
http
{
    include mime.types;
    default_type application/octet-stream;
    server_names_hash_bucket_size 3526;
    server_names_hash_max_size 4096;
    log_format combined_realip '$remote_addr $http_x_forwarded_for [$time_local]'
    ' $host "$request_uri" $status'
    ' "$http_referer" "$http_user_agent"';
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 30;
    client_header_timeout 3m;
    client_body_timeout 3m;
    send_timeout 3m;
    connection_pool_size 256;
    client_header_buffer_size 1k;
    large_client_header_buffers 8 4k;
    request_pool_size 4k;
    output_buffers 4 32k;
    postpone_output 1460;
    client_max_body_size 10m;
    client_body_buffer_size 256k;
    client_body_temp_path /usr/local/nginx/client_body_temp;
    proxy_temp_path /usr/local/nginx/proxy_temp;
    fastcgi_temp_path /usr/local/nginx/fastcgi_temp;
    fastcgi_intercept_errors on;
    tcp_nodelay on;
    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 8k;
    gzip_comp_level 5;
    gzip_http_version 1.1;
    gzip_types text/plain application/x-javascript text/css text/htm 
    application/xml;
    server
    {
        listen 80;
        server_name localhost;
        index index.html index.htm index.php;
        root /usr/local/nginx/html;
        location ~ \.php$ 
        {
            include fastcgi_params;
            fastcgi_pass unix:/tmp/php-fcgi.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME /usr/local/nginx/html$fastcgi_script_name;
        }    
    }
}

4.6 检查配置文件语法错误并启动服务
/usr/local/nginx/sbin/nginx -t
service nginx start

4.7 检查服务是否启动
[root@localhost src]# netstat -lntp |grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      20899/nginx: master 

————————————————————————————————————————Ngnix安装完成——————————————————————————————————————————————

————————————————————————————————————————LNMP搭建全部完成————————————————————————————————————————————


[Step 5]测试LNMP环境
在/usr/local/nginx/html/目录中添加一个test.php 的文件:
[root@localhost php-5.6.30]# vim /usr/local/nginx/html/test.php 
内容如下:

<?php
        phpinfo();
?>

通过客户端访问LNMP服务器（http://IP地址/test.php）,出现 PHP Version 5.6.30页面，证明LNMP已经搭建完成。

————————————————————————————————————————LNMP测试完成————————————————————————————————————————————


[Step 6]安装Wordpress
6.1 解压并复制文件
进入下载目录并解压
cd /usr/local/src
tar -zxvf wordpress-4.9.4-zh_CN.tar.gz 


rm -rf /usr/local/nginx/html/*
cp -a /usr/local/src/wordpress/* /usr/local/nginx/html/


6.2 在mysql中设置wordpress的账号密码,数据库.
登陆mysql.
/usr/local/mysql/bin/mysql -uroot -p

设置用户密码权限等等
mysql> GRANT ALL ON wordpress.* TO 'wpuser'@'localhost' IDENTIFIED BY 'wppass';  #设置用户密码权限


mysql> CREATE DATABASE wordpress;  #创建wordpress数据库

6.3 复制配置文件模版并配置
cd /usr/local/nginx/html/
cp wp-config-sample.php wp-config.php  #配置文件名称不要改其他的,会报错

vim wp-config.php

<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the
 * installation. You don't have to use the web site, you can
 * copy this file to "wp-config.php" and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * MySQL settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://codex.wordpress.org/Editing_wp-config.php
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wpuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'wppass' );

/** MySQL hostname */
define( 'DB_HOST', '127.0.0.1' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication Unique Keys and Salts.
 *
 * Change these to different unique phrases!
 * You can generate these using the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}
 * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'CUZ.=A)B?T(#patCC?t.+.N:z!V3[n|76<[{r>XMl@%g3&6?23(I:9r8Qv(7~Q}~' );
define( 'SECURE_AUTH_KEY',  '{5/5_C Vz.cMA3]pWbM* /~e%Ftx)#D$9BudSw|OOXF,{CDq|FGK/|wjcYK-t~RL' );
define( 'LOGGED_IN_KEY',    'Y2jy#L4{9oq3RcHMWWy!eXp(j+l?=^7}^(bs]}8c>aNu!j+GyUrY+%2^4+-#j7GM' );
define( 'NONCE_KEY',        'liRkM^4 9ZzS0zn^m:vHb?$xhAw1ABSCQXqt+ yjJ$cu0|W!_8a|Z%<r?o1:gZq6' );
define( 'AUTH_SALT',        'B<JO4G0YnVv!vBcKv&n;!5-RIibrb Lhm)p=bm[,#5QQ>&#|aUzERn,D@ILxq>m|' );
define( 'SECURE_AUTH_SALT', 'JIig]Ni,0_-`:9_m4+%[emGJ9vjaO9bG,j>V%%`(F#+%b)[bpB6&B.yOQ9?}(,{u' );
define( 'LOGGED_IN_SALT',   'EqYRhiY}<bQB7^>1M1::R_q/P$=F`o)NODHX8I^yrW5I7-]7x+g%_-!N&>8|6vy3' );
define( 'NONCE_SALT',       '?BK.rbD+wwMI|jA:>%.53jk1Sl]1@-VBq&l]XQ*sh?2(?LCD8Euwu7lcXhJekvTY' );

/**#@-*/

/**
 * WordPress Database Table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the Codex.
 *
 * @link https://codex.wordpress.org/Debugging_in_WordPress
 */
define( 'WP_DEBUG', false );

/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', dirname( __FILE__ ) . '/' );
}

/** Sets up WordPress vars and included files. */
require_once( ABSPATH . 'wp-settings.php' );

6.4 浏览器登陆wordpres.
http://IP or 域名，即可打开wordpress安装页面。















