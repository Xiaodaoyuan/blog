---
title: nginx详解
date: 2018-10-06 19:00:44
categories: 其他
tags: nginx
---

### 1、概述    
　　nginx是一个跨平台的高性能web服务器。目前我们主要使用nginx用来做反向代理服务器。  
　　为什么选择nginx：1、更快；2、高扩展性；3、高可靠性；4、低内存消耗；5、单机支持10w以上并发连接；6、热部署。
<!--more-->
---

### 2、安装和启动
**2.1、编译安装**  
　　从nginx官网获取nginx的源码，然后将源码解压到准备好的目录。然后进入安装目录执行三行代码：./configure、make、make install。  
　　configure命令做了大量的“幕后”工作，包括检测操作系统内核和已经安装的软件，参数的解析，中间目录的生成以及根据各种参数生成一些C源码文件、Makefile文件等。  
　　make命令根据configure命令生成的Makefile文件编译Nginx工程，并生成目标文件、最终的二进制文件。  
　　make install命令根据configure执行时的参数将Nginx部署到指定的安装目录，包括相关目录的建立和二进制文件、配置文件的复制。  
**2.2、命令行控制**
- **默认方式启动：**./nginx 会使用默认路径下的conf文件。
- **另指定配置文件启动：**./nginx -c /tmp/nginx.conf 使用-c参数可以指定配置文件。
- **另指定安装目录启动：**./nginx -p /usr/local/nginx 使用-p参数指定nginx的安装目录。
- **测试配置信息的正确性：**./nginx -t 使用-t参数测试配置文件是否正确。
- **在测试阶段不输出非错误信息：**./nginx -t -q 测试配置选项时，使用-q参数可以不把error级别以下的信息输出到屏幕。
- **显示版本信息：**./nginx -v 使用-v参数显示Nginx的版本信息。
- **快速停止服务：**./nginx -s stop       可以强制停止服务。nginx程序通过nginx.pid文件得到master进程的进程ID，再向master进程发送TERM信号快速的关闭服务。
- **优雅停止服务：**./nginx -s quit  这个会先关闭端口，停止接受新的连接，然后将当前的任务全部处理完再关闭服务。
- **运行中重读配置文件并生效：**./nginx -s reload  可以使运行中的nginx重新加载配置文件。
---

### 3、服务架构
**3.1、模块化架构**  
　　nginx其实就是由很多模块组成，每个模块实现特定的功能。主要分为核心模块、标准http模块、可选http模块、邮件模块以及第三方模块等五大类。  
**3.2、web请求处理机制**  
　　实现并行处理请求工作主要有三种方式：多进程方式、多线程方式、异步方式。nginx主要采用的是多进程机制和异步机制，异步机制使用的是异步非阻塞。  
**3.3、事件驱动模型**  
　　异步IO的实现机制主要有两种：一是工作进程不停的查询IO是否完成；二是IO调用完成后主动通知工作进程。nginx使用的是第二种，也就是事件驱动模型。  事件收集器-->事件发送器-->事件处理器  
　　常用的事件驱动模型库有：select库、poll库、epoll库。
- select库：首先创建读、写、异常三类事件描述符集合，其次调用底层select函数，等待事件发生，然后轮询三个集合中的每一个事件描述符，如果有事件发生就进行处理。
- poll库：poll库和select库的基本工作方式是相同的，主要区别在于select库创建了三个集合，而poll库只需要创建一个集合，是select库的优化实现。
- epoll库：epoll的实现方式是把描述符列表的管理交给内核负责，一旦有某种事件发生，内核就把事件发生的描述符列表通知给进程，这样避免了轮询整个描述符列表，所以它比select库和poll库更高效。

**3.4、设计架构概览**  
　　nginx的主要架构是由一个主进程(master process)，多个工作进程(worker process)组成。主进程主要进行nginx配置文件解析、数据结构初始化、模块配置和注册、信号处理、网络监听生成、工作进程生成和管理等工作，工作进程主要进行进程初始化、模块调用和请求处理等工作，是nginx服务器提供服务的主体。  

---

### 4.配置
下面是[nginx官网](http://nginx.org/en/docs/)基本配置的一个完整例子，更多的配置可以去官网查询相关文档。

```
#用户（组）
user  www www;
#工作进程数
worker_processes  2;
#进程PID存放路径
pid /var/run/nginx.pid;
#错误日志存放路径
# [ debug | info | notice | warn | error | crit ]
error_log  /var/log/nginx.error_log  info;

#events块
events {
    #配置最大连接数
    worker_connections   2000;
    #事件驱动模型选择
    # use [kqueue|epoll|/dev/poll|select|poll];
    use kqueue;
}

#http块
http {
    #定义MIME-Type
    include       conf/mime.types;
    default_type  application/octet-stream;

    #日志格式
    log_format main      '$remote_addr - $remote_user [$time_local] '
                         '"$request" $status $bytes_sent '
                         '"$http_referer" "$http_user_agent" '
                         '"$gzip_ratio"';

    log_format download  '$remote_addr - $remote_user [$time_local] '
                         '"$request" $status $bytes_sent '
                         '"$http_referer" "$http_user_agent" '
                         '"$http_range" "$sent_http_content_range"';

    client_header_timeout  3m;
    client_body_timeout    3m;
    send_timeout           3m;

    client_header_buffer_size    1k;
    large_client_header_buffers  4 4k;

    gzip on;
    gzip_min_length  1100;
    gzip_buffers     4 8k;
    gzip_types       text/plain;

    output_buffers   1 32k;
    postpone_output  1460;

    sendfile         on;
    tcp_nopush       on;
    tcp_nodelay      on;
    send_lowat       12000;

    keepalive_timeout  75 20;

    #lingering_time     30;
    #lingering_timeout  10;
    #reset_timedout_connection  on;


    server {
        listen        one.example.com;
        server_name   one.example.com  www.one.example.com;

        access_log   /var/log/nginx.access_log  main;

        location / {
            proxy_pass         http://127.0.0.1/;
            proxy_redirect     off;

            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            #proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;

            client_max_body_size       10m;
            client_body_buffer_size    128k;

            client_body_temp_path      /var/nginx/client_body_temp;

            proxy_connect_timeout      70;
            proxy_send_timeout         90;
            proxy_read_timeout         90;
            proxy_send_lowat           12000;

            proxy_buffer_size          4k;
            proxy_buffers              4 32k;
            proxy_busy_buffers_size    64k;
            proxy_temp_file_write_size 64k;

            proxy_temp_path            /var/nginx/proxy_temp;

            charset  koi8-r;
        }

        error_page  404  /404.html;

        location = /404.html {
            root  /spool/www;
        }

        location /old_stuff/ {
            rewrite   ^/old_stuff/(.*)$  /new_stuff/$1  permanent;
        }

        location /download/ {

            valid_referers  none  blocked  server_names  *.example.com;

            if ($invalid_referer) {
                #rewrite   ^/   http://www.example.com/;
                return   403;
            }

            #rewrite_log  on;

            # rewrite /download/*/mp3/*.any_ext to /download/*/mp3/*.mp3
            rewrite ^/(download/.*)/mp3/(.*)\..*$
                    /$1/mp3/$2.mp3                   break;

            root         /spool/www;
            #autoindex    on;
            access_log   /var/log/nginx-download.access_log  download;
        }

        location ~* \.(jpg|jpeg|gif)$ {
            root         /spool/www;
            access_log   off;
            expires      30d;
        }
    }
}
```
---

### 5.反向代理与负载均衡
**正向代理和反向代理：** 简单说，正向代理用来让局域网客户机接入外网以访问外网资源，反向代理用来让外网的客户端接入局域网中的站点以访问站点中的资源。  
- 正向代理
```
server {  
   resolver 192.168.1.1; #指定DNS服务器IP地址  
   listen 8080;  
   location / {  
       proxy_pass http://$http_host$request_uri; #设定代理服务器的协议和地址  
   }  
} 
```
- 反向代理

```
server {
   listen       80;
   location /demo {
       proxy_pass http://127.0.0.1:8081;
   }
   location /demo1 {
       proxy_pass http://127.0.0.1:8082;
   }
}
```
**负载均衡**  
　　nginx是一个非常高效的HTTP负载均衡器，将流量分配给多个应用服务器，并通过nginx提高web应用程序的性能、可伸缩性和可靠性。  
　　nginx负载均衡使用的指令主要有upstream和proxy_pass，upstream块定义一个后端集群，proxy_pass用于location块中，表示对于所有符合要求的request交给某个集群处理。   
　　常见的由以下几种实现负载均衡的方式。
- 轮询（round-robin）：默认方式

```
http {
   upstream backend {
       server srv1.example.com;
       server srv2.example.com;
       server srv3.example.com;
   }
   server {
       listen 80;
       location / {
           proxy_pass http://backend;
       }
   }
}
```

- 加权轮询（weight-round-robin）

```
upstream backend {
   server srv1.example.com weight=3;
   server srv2.example.com weight=2;
   server srv3.example.com;
}

```

- 最少连接（least-connected）

```
upstream backend {
   least_conn;
   server srv1.example.com;
   server srv2.example.com;
   server srv3.example.com;
}

```

- ip-hash：每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。

```
upstream backend {
   ip_hash;
   server srv1.example.com;
   server srv2.example.com;
   server srv3.example.com;
}
```

- fair（第三方）: 按后端服务器的响应时间来分配请求，响应时间短的优先分配

```
upstream backend {
   fair;
   server srv1.example.com;
   server srv2.example.com;
   server srv3.example.com;
}
```








