
### A glipse of Nginx

    Nginx ("engine x") 是一个高性能的 HTTP 和 反向代理 服务器，也是一个 IMAP/POP3/SMTP 代理服务器。 Nginx 是由 Igor Sysoev 为俄罗斯访问量第二的 Rambler.ru 站点开发的，第一个公开版本0.1.0发布于2004年10月4日。其将源代码以类BSD许可证的形式发布，因它的稳定性、丰富的功能集、和低系统资源的消耗而闻名。

### 1. 安装Nginx（源码编译安装，平台为rhel6.5.x86_64）
1.1下载源码包并解压（尽量选择稳定版本）

    [root@lockey ~]# wget http://nginx.org/download/nginx-1.12.1.tar.gz
    [root@lockey ~]# tar zxvf nginx-1.12.1.tar.gz

1.2 编译前的配置

编译安装nginx的时候为了安全起见需要在源代码文件中把版本号注释掉，这是为了防止针对特定版本的恶意攻击

    [root@lockey ~]#vim /root/nginx-1.12.1/src/core/nginx.h
    #define NGINX_VER          "nginx/" // NGINX_VERSION

关闭编译时的调试模式，这样编译得到的源码包的大小会减少很多

     [root@lockey ~]#cd /root/nginx-1.12.1/auto/cc
     [root@lockey ~]#vim gcc
      # debug
      #CFLAGS="$CFLAGS -g"


###配置

    ./configure --user=www --group=www --prefix=/usr/local/nginx --with-file-aio  --with-threads --with-http_stub_status_module --with-http_ssl_module --with-http_realip_module

参数解释：

    #--prefix=/usr/local/nginx指定安装路径

    #--with-http_stub_status_module开启Nginx自带状态检测模块

    #--with-http_ssl_module开启https模块

    #--with-file-aio 开启文件AIO支持

    #--with-threads 启用线程池支持

配置过程中可能出现的问题以及解决：

    缺少PCRE库的支持
    解决：yum install pcre-devel -y

    缺少openssl支持：
    解决：yum install openssl-devel -y

1.3 编译和安装

    [root@lockey nginx-1.12.1]# make && make install

1.4 服务的启动以及测试

注意：一般对配置文件/usr/local/nginx/conf/nginx.conf做了修改后需要首先运行以下命令检查配置是否有语法错误然后再进行服务的停启动作：

    /usr/local/nginx/sbin/./nginx -t

     服务启动方法一：/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
     启动方法二：cd /usr/local/nginx/sbin && ./nginx

 测试：在浏览器中输入http://ip:80能够看到Nginx的欢迎界面则基本的配置就算是成功了

     关于服务的停止/usr/local/nginx/sbin/nginx -s stop
     服务的配置重新加载：/usr/local/nginx/sbin/nginx -s reload

### 2. 配置文件基本项解读

    user  www www;
    #运行的用户和组

    worker_processes  2;
    worker_cpu_affinity 01 10;#两核配置如此
    #将进行数和CPU做绑定，可以通过lscpu查看CPU的核数
    #worker_cpu_affinity 001 010 011 100;四核配置如此
    #要注意 cpu 不均衡的问题，原因很多的：磁盘、文件I/O等

    #error_log  logs/error.log;
    #error_log  logs/error.log  notice;
    #error_log  logs/error.log  info;

    #pid        logs/nginx.pid;


    events {
        worker_connections  65535;
    }
    #连接量修改之后需要修改系统限制配置文件 vim /etc/security/limits.conf,添加内容如下：
    #nginx   -   nofile    65535
    http {
        #设定mime类型,类型由mime.type文件定义
        include    mime.types;
        default_type  application/octet-stream;
        #设定日志格式
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  logs/access.log  main;

        #sendfile 指令指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，
        #对于普通应用，必须设为 on,
        #如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，
        #以平衡磁盘与网络I/O处理速度，降低系统的uptime.
        sendfile     on;
        #tcp_nopush     on;

        #连接超时时间
        #keepalive_timeout  0;
        keepalive_timeout  65;
        tcp_nodelay     on;

        #开启gzip压缩
        gzip  on;
        gzip_disable "MSIE [1-6].";

        #设定请求缓冲
        client_header_buffer_size    128k;
        large_client_header_buffers  4 128k;


        #设定虚拟主机配置
        server {
            #侦听80端口
            listen    80;
            #定义使用 www.nginx.cn访问
            server_name  www.nginx.cn;

            #定义服务器的默认网站根目录位置
            root html;

            #设定本虚拟主机的访问日志
            access_log  logs/nginx.access.log  main;

            #默认请求
            location / {

                #定义首页索引文件的名称
                index index.php index.html index.htm;   

            }

            # 定义错误提示页面
            error_page   500 502 503 504 /50x.html;
            location = /50x.html {
            }

            #静态文件，nginx自己处理
            location ~ ^/(images|javascript|js|css|flash|media|static)/ {

                #过期30天，静态文件不怎么更新，过期可以设大一点，
                #如果频繁更新，则可以设置得小一点。
                expires 30d;
            }

            #PHP 脚本请求全部转发到 FastCGI处理. 使用FastCGI默认配置.
            location ~ .php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
                include fastcgi_params;
            }

            #禁止访问 .htxxx 文件
                location ~ /.ht {
                deny all;
            }

        }
    }

### 3. Nginx配置虚拟主机

  虚拟主机是将一台服务器主机分成一台台“虚拟”的主机，每台虚拟主机都可以具有独立的域名，具有完整的Intemet服务器功能（WWW、FTP、Email等），同一台主机上的虚拟主机之间是完全独立的。从网站访问者来看，每一台虚拟主机和一台独立的主机完全一样。

  利用虚拟主机，不必为每个要运行的网站提供一台单独的Nginx服务器或单独运行一组Nginx进程。虚拟主机提供了在同一台服务器、同一组Nginx进程上运行多个网站的功能。
  Nginx实现虚拟主机有以下三种方法

  Nginx基于IP的虚拟主机配置：同一台主机的不同ip对应不同的网站，ip占位在server_name
  Nginx基于端口的虚拟主机配置：同一主机的不同端口对应不同网站
  Nginx基于域名的虚拟主机配置：每一个server_name对应一个网站的发布目录

 基于域名的虚拟主机配置示例如下：
 vim /usr/local/nginx/conf/nginx.conf

     server {
     #虚拟主机 1
          listen       80;
          server_name  www.lockey.com;
          #主机名称，也即域名
          #charset koi8-r;

          #access_log  logs/host.access.log  main;

          location / {
              root   /lockey;
              #虚拟主机的发布目录
              index index.html index.htm;
          }
      }

      server {
      #虚拟主机2
          listen       80;
          server_name  www.halo.com;
          #虚拟主机2的主机名，也即域名
          #charset koi8-r;

          #access_log  logs/host.access.log  main;

          location / {
              root   /halo;
              #虚拟主机2的网站发布目录
              index index.html index.htm;
          }
      }
配置完成后需要重载配置，然后进行访问：

    /usr/local/nginx/sbin/nginx -s reload


### 4. Nginx支持https配置

步骤1：生成密钥

    [root@server653 ~]# cd /etc/pki/tls/certs/
    [root@server653 certs]# make cert.pem
    Country Name (2 letter code) [XX]:cn
    State or Province Name (full name) []:shaanxi
    Locality Name (eg, city) [Default City]:xi'an
    Organization Name (eg, company) [Default Company Ltd]:lockey
    Organizational Unit Name (eg, section) []:linux
    Common Name (eg, your name or your server's hostname) []:server653
    Email Address []:lockey@123.com
    [root@server653 certs]# ll cert.pem
    -rw-------. 1 root root 3092 9月  13 10:47 cert.pem
    [root@server653 certs]# cp cert.pem /usr/local/nginx/conf/

步骤2：配置nginx支持https

    [root@server653 ~]# vim /usr/local/nginx/conf/nginx.conf

     # HTTPS server
        server {
            listen       443 ssl;
            server_name  www.lockey.com;###指定要进行加密的域

            ssl_certificate      cert.pem;#####认证与加密文件
            ssl_certificate_key  cert.pem;#####

            ssl_session_cache    shared:SSL:1m;
            ssl_session_timeout  5m;

            ssl_ciphers  HIGH:!aNULL:!MD5;
            ssl_prefer_server_ciphers  on;

            location / {
                root   html;
                index  index.html index.htm;
            }
        }

配置完成后需要重载配置，然后进行访问（https://www.lockey.com）：

    /usr/local/nginx/sbin/nginx -s reload

### 5. Nginx访问认证配置

首先生成认证文件（注意路径）

    [root@server653 conf]# touch passwd.db
    [root@server653 conf]# htpasswd -c -b passwd.db lockey lockey
    Adding password for user lockey

然后配置需要认证的网站，以www.lockey.com为例：

    server {
          listen       80;
          server_name  www.lockey.com;

          #charset koi8-r;

          #access_log  logs/host.access.log  main;

          location / {
              root   html;
              auth_basic 'Welcome to lockey.com!';
        #认证欢迎语
              auth_basic_user_file /usr/local/nginx/conf/passwd.db;
        #认证文件的存储位置           
        index index.php index.html index.htm;
          }
配置完成后需要重载配置，然后进行访问（https://www.lockey.com），会提示需要输入用户名和密码的：

    /usr/local/nginx/sbin/nginx -s reload
  
### 6.反向代理
#注意172.25.22.1，172.25.22.2上边都配置了网络服务并做了域名解析

    upstream lockey{
    server 172.25.22.1:80 weight=2;
    server 172.25.22.2:80;
    server 127.0.0.1:80 backup;#作为一个无法提供服务的友好提示界面
    #可以加权重weight=number;
    #算法默认是轮询，还有ip_hash等
    }


    location / {
           proxy_pass http://lockey;#设置代理访问查询，和upstream lockey对应
           #root   html;
        #auth_basic 'Welcome to lockey.com!';
        #auth_basic_user_file /usr/local/nginx/conf/passwd.db;
        #index index.php index.html index.htm;
    }
配置完成后需要重载配置，然后进行访问测试，可以看到访问到的内容会不停的变化：

  /usr/local/nginx/sbin/nginx -s reload
  
### 7. Nginx重定向（Nginx https重定向）

    server {
            listen       80;
            server_name  www.halo.com;
            rewrite ^(.*)$ https://www.halo.com$1 permanent;#redirect分为
        location / {
          root   /halo;
          index  index.html index.htm;
             }
        }

### 8. 资源重定向
以存储在本地的图片资源通过网络获取为例：

    location /images {
                    alias /images;#当在网页中加载一张图片时可以这样写<img src="/images/pic.png">，实际获取
                的是根目录下的images目录中的图片，而不是网站根目录下的。
                }
