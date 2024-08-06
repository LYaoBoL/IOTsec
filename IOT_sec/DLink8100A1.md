# 基础信息分析
binwalk解压文件，查看一下/bin/busybox
![[Pasted image 20240805151556.png]]
发现为32位小端序的mips架构的

firmwalker-pro-max查看其web服务为jhttpd
![[Pasted image 20240805151657.png]]
```
init 是系统启动的第一个用户空间进程，负责初始化系统和启动其他进程。因此，它是整个启动流程的关键
```
看到文件夹下有init二进制文件，查看了一下，发现其指向了sbin下rc开头的文件
![[Pasted image 20240805152641.png]]
去sbin下面查看rc开头的文件，发现其最后都指向了rc这个二进制文件
![[Pasted image 20240805152933.png]]
拖进IDA分析一下，发现在init_main函数中调用了start_jhttpd函数，将jhttpd服务启动
![[Pasted image 20240805153446.png]]
![[Pasted image 20240805153652.png]]

按照我们正常分析固件的思路，一般去分析启动项，都会去/etc/rcS(这里是/etc_ro/rcS)中，可是去看发现，rcS中启动了goahead
![[Pasted image 20240805155555.png]]
这又超出了我的认知，于是想着FirmAE模拟进shell中查看进程，看一下到底是什么服务
![[Pasted image 20240805155734.png]]
确实是jhttpd服务，确实是没有goahead服务，那么rcS这里写的goahead又是什么呢，很奇怪
那么我们把jhttpd给kill掉，看到底是哪里启动了

kill掉后我们先去运行我们常规分析思路的rcS
![[Pasted image 20240805160257.png]]
发现从goahead就开始报错了，就已经找不到了，正如我们去文件系统中搜索goahead一样，就根本不存在goahead，那么服务就根本也起不来的呀
由于init->/sbin/rc，故我们去执行init，执行完成后ps了一下确实jhttpd启动成功了
![[Pasted image 20240805162335.png]]

# 漏洞复现
其实我们发现，在固件的文件系统中并没有发现web的界面存在，也是头一次遇到这样子的情况，但我们在jhttpd这个二进制文件中，用IDA看会有很多以_htm和_asp结尾的函
![[Pasted image 20240805195235.png]]
![[Pasted image 20240805200526.png]]
于是我们猜测，这固件的web端应该是集成在jhttpd的二进制文件中
那我们分析的时候，就应该是去搜索system危险函数，去分析哪里没有过滤可以输入的点
![[Pasted image 20240805201130.png]]
这里我们搜索查询到在msp_info_htm函数中，参数flag=cmd&cmd=%s也就是我们要执行命令的点，拼接起来执行了system
于是我们带着参数flag=cmd&cmd=反引号ps>jiawen.txt反引号访问这个界面
http://192.168.0.1/msp_info.htm?flag=cmd&cmd=`ps%3Ejiawen.txt`· 这里下载了一个文件
抓包去看了一下，是有回显的
![[Pasted image 20240805194744.png]]
进入shell查看有没有jiawen.txt的文件
![[Pasted image 20240805210842.png]]
说明命令执行成功了


main->httpd_poll->httpd_do_recv->httpd_dowith_get->httpd_send_file->msp_info_htm

![[Pasted image 20240805214603.png]]
![[Pasted image 20240805215042.png]]
![[Pasted image 20240805215204.png]]