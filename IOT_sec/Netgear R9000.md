# 基础信息分析
先binwalk一下，看一下基础信息
![[Pasted image 20240731183120.png]]
发现是ARM架构的小端序，文件系统是squashfs
firmwalker-pro-max 扫描发现起的是uhttpd服务
![[Pasted image 20240731200915.png]]
# 模拟固件
FirmAE模拟不起来，于是就系统模拟一下，先配置一下网卡
![[Pasted image 20240731222557.png]]
![[Pasted image 20240731222539.png]]
```
sudo qemu-system-arm -M vexpress-a9 -kernel vmlinuz-3.2.0-4-vexpress -initrd initrd.img-3.2.0-4-vexpress -drive if=sd,file=debian_wheezy_armhf_standard.qcow2 -append "root=/dev/mmcblk0p2 console=ttyAMA0" -net nic -net tap,ifname=tap0,script=no,downscript=no -nographic
#vexpress-a9开发板
#加载的Linux内核
#initrd.img-3.2.0-4-vexpre ss是内核启动时加载的初始文件系统
```
模拟起来，账号密码均为root
![[Pasted image 20240731203224.png]]
记得在system虚拟机中也配置一下网卡
![[Pasted image 20240731222908.png]]
配置好了记得测试一下连通性
![[Pasted image 20240731222941.png]]
把固件打包，发送给我们模拟的虚拟机中
![[Pasted image 20240731202812.png]]
通过SimpleHTTPServer
![[Pasted image 20240731204709.png]]
解压出来再赋予权限
![[Pasted image 20240731205026.png]]
```
cd /squashfs-root
mount --bind /proc proc
# proc目录是一个虚拟文件系统，可以为linux用户空间和内核空间提供交互
mount --bind /dev dev
# /dev/下的设备是通过创建设备节点生成的，用户通过此设备节点来访问内核里的驱动
chroot . sh

因为 `chroot` 会导致无法在隔离的文件系统中访问原本的 `/proc`和 `/dev` 目录，这里利用 `mount` 命令将 `qemu-system-armhf` 的 `proc` 和 `dev` 目录挂在到 `squashfs-root` 中，并更换根目录为 `squashfs-root`。
```
![[Pasted image 20240731205815.png]]
模拟是终于起来了，接下来该看web端该怎么启动的了
之前firmwalker-pro-max也提到，这里的服务是uhttpd，于是去查找一下uhttpd是怎么启动的
看了一下启动项文件中存在一个uhttpd.sh脚本，看了一下
![[Pasted image 20240731212931.png]]
是在/www/cgi-bin/uhttpd.sh start了，再去看一下这里
![[Pasted image 20240731213232.png]]
通过一条命令启动了uhttpd，把参数打印出来看一下是什么
![[Pasted image 20240731225019.png]]
先分析一下命令要干什么
```
-h /www： 指定文档根目录。服务器将从这个目录提供静态文件。/www 是文档根目录的路径。
-r R9000：设置主机名。这个参数通常用来定义服务器的标识。R9000 是主机名。
-x /cgi-bin： 设置CGI脚本目录。服务器将从这个目录执行CGI脚本。 /cgi-bin 是CGI脚本目录的路径。
-t 70： 设置连接超时时间（秒）。如果连接在指定时间内没有活动，服务器将关闭连接。 70 是超时时间，单位为秒。
-p 0.0.0.0:80： 设置HTTP监听地址和端口。服务器将监听指定的地址和端口来处理HTTP请求。 0.0.0.0 表示监听所有网络接口，80 是HTTP端口。 
-C /etc/uhttpd.crt： 指定SSL证书文件的路径。用于HTTPS连接。 /etc/uhttpd.crt 是SSL证书文件的路径。 
-K /etc/uhttpd.key： 指定SSL密钥文件的路径。用于HTTPS连接。 /etc/uhttpd.key 是SSL密钥文件的路径。 
-s 0.0.0.0:443： 设置HTTPS监听地址和端口。服务器将监听指定的地址和端口来处理HTTPS请求。 0.0.0.0 表示监听所有网络接口，443 是HTTPS端口。
```
我们不需要https ,所以只需要执行如下：
/usr/sbin/uhttpd -h /www -r R9000 -x /cgi-bin -t 70 -p 0.0.0.0:80
拼接命令执行，启动uhttpd，ps看一下执行成功了，但是有两个不知道为什么了
![[Pasted image 20240731225845.png]]
去访问web端的时候发现，404
![[Pasted image 20240731225925.png]]
发现访问/www/目录下面的htm访问不了，但是js、gif等文件还是可以访问的到的，说明服务还是起来了，但是还是有些问题
![[Pasted image 20240731233056.png]]
没办法查看一下错误信息吧，在IDA里搜索不到，在文件系统中发现是个变量，再搜索一下变量名
![[Pasted image 20240801100044.png]]
![[Pasted image 20240801100939.png]]
发现存在于net-cgi这个二进制文件中，搜索bad_file通过交叉引用定位到了
![[Pasted image 20240801103418.png]]
于是查看一下LABEL_209是从哪里调用了
![[Pasted image 20240801103527.png]]
主要判断了v86，v86又来自v134，再追溯一下v134从何而来
![[Pasted image 20240801104154.png]]
v134只有在这里赋了值，取了v8，v8又是根据获取的环境变量
```
CONTENT_LENGTH 是一个环境变量，用于表示 HTTP 请求主体的长度（以字节为单位）
```
要想不跳到209显示404，故v8不能为0
但仔细一想，/www目录下的文件获取到的CONTENT_LENGTH应该均不为0，那为什么访问不了呢，我们忽略了上面的一个点
![[Pasted image 20240801122432.png]]
在这里用自定义的函数跟.htm做了对比，包含.htm就会被过滤掉，所以访问不了