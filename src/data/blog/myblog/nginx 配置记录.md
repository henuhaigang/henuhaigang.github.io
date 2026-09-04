
---
layout:     post
title:      nginx 配置速查
pubDatetime: 2026-08-31T04:59:04.866Z
modDatetime: 2026-08-31T04:59:04.866Z
header-img: img/post-bg-debug.png
catalog: true
author: henuhaigang
tags:
    - 终端
    - nginx
description: nginx配置详解参考
---
Nginx 的核心配置文件通常为 nginx.conf，其结构清晰，主要由‌全局块（Main）‌、‌Events 块‌、‌HTTP 块‌（包含 Server 和 Location）组成。

以下是一份详细且通用的 Nginx 配置示例，涵盖了性能优化、反向代理、负载均衡、日志管理及动静分离等核心功能，并附带逐行详细说明。

1. Nginx 配置文件结构概览
nginx
# ================= 全局块 (Main Context) =================
# 指定运行 Nginx 的用户和用户组，提高安全性
user www-data;

# 工作进程数，通常设置为 CPU 核心数或 auto（自动检测）
worker_processes auto;

# 错误日志路径及级别 (debug, info, notice, warn, error, crit, alert, emerg)
error_log /var/log/nginx/error.log warn;

# PID 文件路径，记录主进程 ID
pid /var/run/nginx.pid;

# 单个 worker 进程最大打开文件描述符数，需配合系统 ulimit 设置
worker_rlimit_nofile 65535;

# ================= Events 块 =================
events {
    # 使用 epoll 事件驱动模型（Linux 下最高效）
    use epoll;

    # 每个 worker 进程允许的最大并发连接数
    # 最大理论连接数 = worker_processes * worker_connections
    worker_connections 1024;

    # 是否允许同时接受多个新连接
    multi_accept on;
}

# ================= HTTP 块 =================
http {
    # 引入 MIME 类型定义文件
    include /etc/nginx/mime.types;
    
    # 默认文件类型，当无法识别文件类型时使用
    default_type application/octet-stream;

    # --- 日志格式定义 ---
    # 定义名为 'main' 的日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    # 访问日志路径及使用的格式
    access_log /var/log/nginx/access.log main;

    # --- 基础性能优化 ---
    # 开启高效文件传输模式 (sendfile)，减少用户态与内核态切换
    sendfile on;
    
    # 配合 sendfile 使用，防止网络阻塞
    tcp_nopush on;
    
    # 禁用 Nagle 算法，减少网络延迟
    tcp_nodelay on;

    # 保持连接超时时间
    keepalive_timeout 65;

    # 客户端请求头缓冲区大小
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;

    # 限制上传文件大小，默认 1m，上传大文件需调大
    client_max_body_size 10m;

    # --- Gzip 压缩配置 ---
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # --- 上游服务器组 (负载均衡) ---
    upstream backend_servers {
        # 负载均衡策略：轮询 (默认)、ip_hash、least_conn 等
        # ip_hash: 根据客户端 IP 哈希分配，解决 Session 问题
        # least_conn: 最少连接数优先
        
        server 192.168.1.10:8080 weight=5; # 权重高，处理更多请求
        server 192.168.1.11:8080 weight=3;
        server 192.168.1.12:8080 backup;   # 备份服务器，仅当其他服务器宕机时启用
    }

    # ================= Server 块 (虚拟主机) =================
    server {
        # 监听端口
        listen 80;
        
        # 服务器域名，支持通配符和正则
        server_name example.com www.example.com;

        # 字符集
        charset utf-8;

        # --- 静态资源根目录 ---
        root /usr/share/nginx/html;
        index index.html index.htm;

        # --- Location 匹配规则 ---
        
        # 1. 精确匹配或前缀匹配：处理静态资源
        location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
            expires 30d; # 浏览器缓存过期时间
            access_log off; # 静态资源通常不记录访问日志以节省 IO
            add_header Cache-Control "public";
        }

        # 2. 反向代理：将动态请求转发给后端服务器
        location /api/ {
            proxy_pass http://backend_servers; # 指向 upstream 定义的组
            
            # 传递真实客户端 IP 给后端
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # 代理超时设置
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # 3. 默认位置
        location / {
            try_files $uri $uri/ /index.html;
        }

        # 4. 错误页面配置
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }

    # ================= HTTPS Server 块 (可选) =================
    # server {
    #     listen 443 ssl http2;
    #     server_name example.com;
    #     
    #     ssl_certificate /etc/nginx/ssl/cert.pem;
    #     ssl_certificate_key /etc/nginx/ssl/key.pem;
    #     
    #     # SSL 优化配置
    #     ssl_protocols TLSv1.2 TLSv1.3;
    #     ssl_ciphers HIGH:!aNULL:!MD5;
    #     ssl_prefer_server_ciphers on;
    #     
    #     location / {
    #         proxy_pass http://backend_servers;
    #     }
    # }
}

2. 核心配置模块详解
A. 全局块 (Main)
worker_processes‌: 决定 Nginx 启动多少个工作进程。设置为 auto 会让 Nginx 自动根据 CPU 核心数调整，这是大多数场景下的最佳实践。
error_log‌: 定义错误日志。级别建议生产环境使用 warn 或 error，调试时使用 info 或 debug。
worker_rlimit_nofile‌: 修改进程能打开的最大文件句柄数。高并发场景下必须调大此值，否则会出现 "Too many open files" 错误。
B. Events 块
use epoll‌: Linux 系统下推荐使用 epoll，它是多路复用 I/O 模型，能高效处理大量并发连接。
worker_connections‌: 单个进程的最大连接数。总并发能力约为 worker_processes * worker_connections。注意不要超过系统限制。
C. HTTP 块
include mime.types‌: 确保 Nginx 能正确识别文件类型（如 .css, .js），从而返回正确的 Content-Type。
sendfile on‌: 开启零拷贝技术，直接在 kernel 空间传输文件，极大提升静态文件服务性能。
gzip‌: 开启压缩可显著减少带宽消耗，提升页面加载速度。gzip_min_length 避免压缩过小的文件（压缩开销可能大于收益）。
upstream‌: 定义后端服务器集群。
weight‌: 权重，数值越大分配的请求越多。
backup‌: 标记为备份节点，只有主节点全部不可用时才生效。
策略‌: 默认为轮询（Round Robin），也可配置 ip_hash（会话保持）或 least_conn（最少连接）。
D. Server 块
listen‌: 指定监听的端口。
server_name‌: 基于域名的虚拟主机配置。Nginx 会根据请求头中的 Host 字段匹配对应的 server 块。
root vs alias‌:
root: 将 location 路径追加到 root 路径后。例如 root /var/www; location /img { ... } 访问 /img/a.jpg 对应文件 /var/www/img/a.jpg。
alias: 替换路径。例如 alias /data/pics; 访问 /img/a.jpg 对应文件 /data/pics/a.jpg。
E. Location 块
匹配优先级‌:
= : 精确匹配，优先级最高。
^~ : 前缀匹配，如果匹配成功，不再检查正则。
~ / ~* : 正则匹配（区分/不区分大小写）。
无前缀修饰 : 普通前缀匹配。
proxy_pass‌: 反向代理的核心指令。注意末尾是否有 /：
proxy_pass http://backend; (无斜杠): URI 原样转发。
proxy_pass http://backend/; (有斜杠): 去掉 location 匹配的部分后转发。
3. 常用管理命令

在修改配置文件后，务必执行以下操作以确保服务稳定：

检查配置语法‌:

bash
nginx -t


如果输出 syntax is ok 和 test is successful，则配置无误。

平滑重载配置‌ (无需停止服务):

bash
nginx -s reload


查看版本及编译参数‌:

bash
nginx -V


停止服务‌:

bash
nginx -s stop  # 快速停止
nginx -s quit  # 优雅停止（处理完当前请求后退出）

4. 最佳实践建议
模块化配置‌: 当站点较多时，不要在 nginx.conf 中堆积所有 server 块。使用 include /etc/nginx/conf.d/*.conf; 将不同域名的配置拆分到独立文件中。
安全加固‌:
隐藏 Nginx 版本号：在 http 块中添加 server_tokens off;。
限制 HTTP 方法：在 location 中只允许必要的 method，如 if ($request_method !~ ^(GET|POST)$) { return 405; }。
日志切割‌: Nginx 本身不支持自动日志切割，建议配合 logrotate 工具定期切割和清理日志文件，防止磁盘占满。
HTTPS 强制跳转‌: 为了安全，建议在监听 80 端口的 server 块中使用 return 301 https://$host$request_uri; 将 HTTP 请求永久重定向到 HTTPS。