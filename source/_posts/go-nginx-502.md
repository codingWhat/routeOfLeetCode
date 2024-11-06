---
title: go_nginx_502问题排查
date: 2024-07-09 16:35:08
tags:
- Go
- Nginx
- 502
- keepalive
---
> 线上巡检发现很多502日志，于是就开始了漫漫debug
<!-- more -->

简单介绍背景
1. 线上服务:
- 容器部署
- http
- Nginx + Go
- 服务耗时基本在100ms左右

2. 已做排查，排除服务不可用导致的502问题
- 服务是否重启
- 容器是否异常、重启
- 磁盘、cpu是否异常



## 问题现场
### 问题1: upstream prematurely closed connection 
在排查nginx日志时发现如下错误
> nginx error log: "upstream prematurely closed connection while reading response header from upstream"

很明显服务主动关闭了连接，httpServer主动关闭连接一般是read/write超时了, 但是查看服务配置发现read/write分别1s/3s, 并且服务逻辑中都有严格的超时控制、没有阻塞逻辑，讲道理不太可能触发，所以这里排除。问题到这里似乎进到死胡同了，这时在看server源码是发现<font color="red">idletimeout</font>这个配置, 如果没有设置默认取read timeout，经google之后发现就是keepalive的timeout。我们知道http/1.1默认都是keepalive的, 如果触发了keepalive timeout, server会主动关闭连接，于是开始抓包分析(如下图)，发现go服务在1s之后主动断开了和nginx的连接。

![img.png](/images/wireshark_502.png)

这里总结下整个请求链路.
首先nginx和upstream server(go 服务)之间会创建多个连接；外部请求进来以后, nginx作为client端，从连接池获取一个连接请求，如果此时刚好这个连接keepalive timeout了那么就会触发502。

问题解决:
1. nginx proxy设置keepalive;
`proxy_http_version 1.1` 、 `proxy_set_header Connection ""`
upstream不需要外部请求Connection控制，直接清空
```
        location / {
                proxy_pass xxxx;
                
                # !!!!! start 
                proxy_http_version 1.1;
                proxy_set_header Connection "";
                 # !!!!! end 
                #proxy_read_timeout     300;    
                #proxy_connect_timeout  300;
                #proxy_set_header X-Real-IP $remote_addr;
                # needed for HTTPS
                # # proxy_set_header X_FORWARDED_PROTO https;
                #proxy_set_header X-Forwarded-For $remote_addr;
                #proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                #proxy_set_header Host $host;
        }
```
2. nginx.conf设置keepalive timeout
这里时nginx和外部请求的keepalive, 如果超过这时间nginx会关闭连接。
```
keepalive_timeout  60s;
```
3. upstream server设置keepalive timeout
```
&http.Server{
		#Addr: addr,
		#Handler:    http.HandlerFunc(ServeHTTP),
		#ReadTimeout:  time.Duration(httpRunner.ReadTimeout) * time.Second,
		#WriteTimeout: time.Duration(httpRunner.WriteTimeout) * time.Second,
		...
		IdleTimeout:  time.Duration(httpRunner.IdleTimeout) * time.Second,
		...
		#ConnState:    httpRunner.connState,
		#ErrorLog:     syslog.New(httpErrorLog{logger}, "", 0),
	}
```
### 问题2: listen backlog 过低
在对服务进行压测时，发现请求如果走nginx会发生阻塞，而直接压测服务却能正常运行，此时发现nginx日志有大量502

问题解决:
1. listen backlog用了默认长度511, listen backlog是长连接队列长度，如果长度过短，容易打满拒绝请求，将backlog长度调大，能进一步提升吞吐。
2. 注意全局长连接队列限制 `/proc/sys/net/core/somaxconn` 也得调整，`nginx backlog` <= `somaxconn`

### 问题3: 暴力清理nginx日志
通过keepalive配置，502问题确实明显改善了，但是突然过了几天，又偶现了502问题，在排查基础资源监控时发现502的时间点，恰好有磁盘和内存空间骤降；
这里定位是因为反向代理的nginx会记录access日志，而我们的服务流量很高access日志容易写满，需要定时清理，清理逻辑：
```
# crontab
echo > /path/access.log
```
这里有个背景说明下:
access文件是会被采集程序访问上报到日志平台。上述直接"echo > " 是可能会导致os.Cache中日志被清理,可能采集程序就会采集不到，出现异常。

改造逻辑: 
- logrotate 10G切割，只保留1个备份文件
- 备份文件会等段时间才被清理(当前10min), 保证采集程序能采集成功

当然也可以自己写逻辑:
1. 按access.log 10g为切割
2. 历史文件不会立即被清理会，等待10min，保证采集程序能采集成功
3. kill -USR1 nginxpid, 命令nginx重新加载配置。
```
file_path="/path/"
log_file="access.log"
#nginx进程id
nginx_pid="/path/nginx.pid "
#单位:G
max_log_size=10
# 备份日志最长存活时间 单位:s
max_log_ttl=300


timestamp=$(date +%s)
log_back_file="$file_path$log_file-bak-$timestamp"
# 获取文件大小（以字节为单位）
file_size=$(stat -c "%s" "$file_path$log_file")
file_size_gb=$(echo "scale=2; $file_size / 1024^3" | bc)
# 判断文件大小是否超过10G
if (( $(echo "$file_size_gb > $max_log_size " | bc -l) )); then
    mv $file_path$log_file  $log_back_file
    cat $nginx_pid | xargs kill -USR1
fi

# 遍历当前目录下的所有文件
for file in "$file_path/$log_file"-bak-*; do
    # 检查文件是否为普通文件并且修改时间超过10分钟
    if [[ -f "$file" && $(($(date +%s) - $(stat -c %Y "$file"))) -gt $max_log_ttl ]]; then
        # 删除文件
        rm "$file"
        echo "已删除文件: $file"
    fi
done
```
### 优化成果
之前每天必复现, 连续一周未收到告警
![img.png](../images/now_502.png)