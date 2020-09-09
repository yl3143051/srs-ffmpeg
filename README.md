### SRS服务器

------

首先上传zip至目录，执行

```shell
unzip SRS-CentOS7-x86_64-3.0.141.zip 
cd SRS-CentOS7-x86_64-3.0.141
sudo bash INSTALL
```

*【注意】可能提示缺少lsb工具类*

```shell
yum install -y redhat-lsb
```

`srs`目录下，启动、停止、重启SRS

```shell
./etc/init.d/srs start
./etc/init.d/srs stop
./etc/init.d/srs restart
```

*【注意】关闭防火墙*

```shell
systemctl stop firewalld
```



### FFmpeg推流

------

1. 设置yum仓库

```shell
sudo yum install https://download1.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm
sudo yum install http://rpmfind.net/linux/epel/7/x86_64/Packages/s/SDL2-2.0.10-1.el7.x86_64.rpm
```

*【注意】当前系统为CentOS8，如果为CentOS7，需要将`rpmfusion-free-release-8.noarch.rpm`改为`rpmfusion-free-release-7.noarch.rpm`*

2. 安装ffmpeg

```shell
yum -y install ffmpeg
```

3. 查看ffmpeg是否安装成功

```shell
ffmpeg -version
```

4. 简单推流

```shell
ffmpeg -re -i /root/video/WuSha.mp4 -c copy -f flv rtmp://192.168.2.178/live/livestream
```



### st-load压力测试

------

解压`st-load-0.2.zip`文件后在其目录下执行

模拟RTMP多路推流

```shell
--
```

模拟客户端拉流

```shell
./objs/st_rtmp_load --clients=30 --url=rtmp://192.168.2.178/live/livestream
```



### SRS控制台

------

可以登录SRS控制台查看当前服务器状态

```
http://ossrs.net/console
```

【连接SRS】IP为服务器IP，端口默认1985

可以使用SRS播放器在线演示

```
http://ossrs.net/srs.release/releases/demo.html
```



### SRS 源站式集群

------

| 服务器        | 类型        |
| ------------- | ----------- |
| 192.168.2.178 | 源站服务器A |
| 192.168.2.200 | 源站服务器B |
| 192.168.2.203 | 边缘服务器A |
| 192.168.2.204 | 边缘服务器B |

在每个服务器下新建文件`conf/server.conf`

```shell
touch server.conf && vi server.conf
```

源站服务器写入如下内容

```shell
listen              1935;
max_connections     1000;
daemon              on;
srs_log_tank        file;
srs_log_level       trace;
srs_log_file        ./objs/server.log;
pid                 ./objs/server.pid;
http_api {
    enabled         on;
    listen          1985;
}
vhost __defaultVhost__ {
    cluster {
        mode            local;
        origin_cluster  on;
        coworkers       192.168.2.200:1985;
    }
}
```

*【注意】源站A的服务器`coworkers`为`192.168.2.200:1985`，源站B的服务器`coworkers`为`192.168.2.178:1985`，互相交叉*

边缘服务器A、B写入如下内容

```shell
listen              1935;
max_connections     1000;
daemon              on;
srs_log_tank        file;
srs_log_level       trace;
srs_log_file        ./objs/server.log;
pid                 ./objs/server.pid;
vhost __defaultVhost__ {
    cluster {
        mode            remote;
        origin          192.168.2.178:1935 192.168.2.200:1935;
    }
}
```

启动SRS后执行

```shell
./objs/srs -c conf/server.conf
```



### SRS Forward搭建小型集群

------

*【注意】此集群转发方式好像只支持转发一路流*

1. ##### 主SRS配置

将以下内容保存到文件`conf/forward.master.conf`

*【注意】文件`conf/forward.master.conf`原来就存在，也可以使用自命名文件*

在`destination`配置分发的服务器IP和端口

```shell
# conf/forward.master.conf
listen              1935;
max_connections     1000;
pid                 ./objs/srs.master.pid;
srs_log_tank        file;
srs_log_file        ./objs/srs.master.log;
vhost __defaultVhost__ {
    forward {
        enabled on;
        destination 192.168.1.2:1936 192.168.1.2:1937 192.168.1.3:1936;
    }
}
```

启动SRS后执行以下内容，主SRS将流转发到备SRS

```shell
./objs/srs -c conf/forward.master.conf
```

2. ##### 备SRS配置

将以下内容保存到新文件`conf/srs.1936.conf` `conf/srs.1937.conf`

```shell
# conf/srs.1936.conf
listen              1936;
pid                 ./objs/srs.1936.pid;
srs_log_tank        file;
srs_log_file        ./objs/srs.1936.log;
vhost __defaultVhost__ {
}
```

```shell
# conf/srs.1937.conf
listen              1937;
pid                 ./objs/srs.1937.pid;
srs_log_tank        file;
srs_log_file        ./objs/srs.1937.log;
vhost __defaultVhost__ {
}
```

启动备SRS后执行以下内容，开始接收主SRS转发的流

```shell
./objs/srs -c conf/srs.1936.conf
./objs/srs -c conf/srs.1937.conf
```



