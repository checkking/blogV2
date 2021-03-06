---
title: "nginx配置管理源码解析"
date: 2017-10-20T21:07:16+08:00
draft: false
---

### 一个典型的nginx.conf配置
```
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent $request_id "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  www.domain.com;
        root /root/com/domain/www;
        index index.html index.htm index.php;
        location ~ \.(jpg|png|gif|js|css|swf|flv|ico)$ {
            expires 12h;
        }

        #charset koi8-r;

        access_log  logs/host.access.log main;
        error_log  logs/host.error.log debug;

        location / {
            if (!-e $request_filename) {
                rewrite ^(.*)$ /index.php?$1 last ;
                break;
            }
        }
        location /test {
            return 200 "Hello World";
        }
        location ~* ^/(doc|logs|app|sys)/ {
            return 403;
        }

        error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;

        location ~ .*\.(php|php5)?$ {
            fastcgi_connect_timeout 300;
            fastcgi_send_timeout 300;
            fastcgi_read_timeout 300;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

        location ^~ /.well-known/ {
            allow all;
        }
    }

    # another virtual host using mix of IP-, name-, and port-based configuration

    server {
        listen       8091;
        server_name  localhost;
        access_log  logs/yaf.access.log;
        root /root/github/yaflearn/Sample;
        index index.html index.htm index.php;

        location ~ \.(jpg|png|gif|js|css|swf|flv|ico)$ {
            expires 12h;
        }

        location / {
            if (!-e $request_filename) {
                rewrite ^(.*)$ /index.php?$1 last ;
                break;
            }
        }

        location ~ .*\.(php|php5)?$ {
            fastcgi_connect_timeout 300;
            fastcgi_send_timeout 300;
            fastcgi_read_timeout 300;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    }
}
```
高度模块化设计是nginx的架构基础。nginx的模块划分图如下：

![Nginx模块划分](https://images-cdn.shimo.im/zBosAtuL3Tg1cr3E/nginx.gif)

而每个模块都有自己的配置项, 所有模块的配置项都在一个配置文件中体现, 每个配置模块有各自的配置指令（Command），比如各个模块常见的指令有：
- ngx_http_module
<br/>
http (ngx_http_core_module)
<br/>
listen (ngx_http_module)
<br/>
server_name (ngx_http_core_module)
<br/>
access_log (ngx_http_log_module)
<br/>
- ngx_event_module
<br />
events (ngx_events_module)
<br />
worker_connections (ngx_event_core_module)
<br />
accept_mutex (ngx_event_core_module)
<br />
worker_aio_requests (ngx_epoll_module)
<br />
- ngx_mail_module
<br />
mail (ngx_mail_module)
<br />
listen (ngx_mail_core_module)
<br />
timeout (ngx_mail_core_module)
<br />
auth_http_header (ngx_mail_auth_http_module)
<br />
pop3_auth (ngx_mail_auth_http_module)
<br />
proxy_timeout (ngx_mail_proxy_module)
<br />

nginx.conf是在nginx启动的时候解析的，解析完成的配置项存放在ngx_cycle_t结构题的conf_ctx中。
```c
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;
    // others
    // ....
}
```
conf_ctx有4个*的原因是它首先指向一个存放指针的数组(核心模块的配置结构体指针)，这个数组中的指针成员同时又指向了另外一个存放指针的数组(子模块的配置结构体指针)。

![所有http模块配置结构体管理](https://images-cdn.shimo.im/BUpKbhD5HOwNCZFH/nginx_http_config.png!thumbnail)

也就是第一层数组存放了各个核心模块的指针，第二层数组存了每个核心模块的子模块的配置结构体的指针。

### 配置解析流程

配置解析的函数入口为`ngx_init_cycle`,  首先准备数据结构。
```c
for (i = 0; ngx_modules[i]; i++) {
    if (ngx_modules[i]->type != NGX_CORE_MODULE) {
        continue;
    }

    module = ngx_modules[i]->ctx;

    if (module->create_conf) {
        rv = module->create_conf(cycle);
        if (rv == NULL) {
            ngx_destroy_pool(pool);
            return NULL;
        }
        cycle->conf_ctx[ngx_modules[i]->index] = rv;
    }
}

conf.ctx = cycle->conf_ctx;
conf.cycle = cycle;
conf.pool = pool;
conf.log = log;
conf.module_type = NGX_CORE_MODULE;
conf.cmd_type = NGX_MAIN_CONF;
```
可以看到首先创建的是核心模块的配置结构体。

接下来是配置解析。

```c
if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
    environ = senv;
    ngx_destroy_cycle_pools(&conf);
    return NULL;
}
```
