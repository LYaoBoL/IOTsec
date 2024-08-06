# 基础信息收集
binwalk解压固件，得到固件为squashfs文件系统，为小端序
![[Pasted image 20240802185800.png]]
firmwalker-pro-max看到web服务可能是boa和goahead
![[Pasted image 20240802190655.png]]
去看了下启动项boa被注释掉了，goahead是直接启动的
![[Pasted image 20240803195017.png]]
是32位MIPS架构的，且没有什么保护
![[Pasted image 20240804035027.png]]
# 模拟分析固件
用FirmAE模拟此固件
firmwalker-pro-max看到web端下面存在一个叫syscmd.htm界面
![[Pasted image 20240802190513.png]]、
看到这种处于尊重当然第一时间访问一下
![[Pasted image 20240804012259.png]]
访问不到去鸡爪搜索了一下错误信息，追溯了一下，应该是打开不了，return了-1
![[Pasted image 20240804040927.png]]
# 漏洞复现动态调试
静态分析出调用链应该如下
main(0x00423C18)->sub_423F90(0x0042423C)->sub_42383C(0x00423A04)->system
![[Pasted image 20240805102249.png]]

FirmAE模拟4，gdb调试，主要调试goahead服务，pid为538
![[Pasted image 20240804015813.png]]
![[Pasted image 20240804015825.png]]
gdb调试命令脚本   -x mips_little，设置mips架构，设置小端序，加载动态库
![[Pasted image 20240804020005.png]]
进入gdb调试，连接192.168.0.1:1337端口
![[Pasted image 20240804020123.png]]
运行exp，准备调试
![[Pasted image 20240804020203.png]]
下断点进行验证
![[Pasted image 20240804020310.png]]
c运行至断点处，发现跳过了1 2断点直接跳到了3断点处（也不知道是为什么，可能是跟没有符号表有关系，也可能是从main进来，先经过了函数sub_42383C再一步步向上返回，因为3断点是函数sub_42383C的开始，发现exp执行的命令已经存在于栈上）
![[Pasted image 20240804023025.png]]
一直n，单步调试，发现v0已经命令拼接完成，后续就执行了命令，清空了寄存器
![[Pasted image 20240804032310.png]]
命令执行成功
![[Pasted image 20240804051505.png]]
# Tips：
发现该漏洞的思路可能是，直接查询了system危险函数的调用，一个个分析至此漏洞处，发现没过滤且可以拼接命令