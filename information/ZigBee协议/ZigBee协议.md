选择channel信道，start开始嗅探

![](image/2b4a3431950e083bbe54e67f72658a02.png)

而网络密钥要再每次设设备与网关重新配对的时候才会发送一个网络密码，通过抓包可以获取
说直白些就是连过一次了就不会再发了，要第一次连

![](image/d97a8a4277eebcbd32d1784ef86c9c9e.png)

wireshrk中编辑->首选项->protocols->edit->添加key
5A6967426565416C6C69616E63653039为链接密钥都是默认的
后者为每个设备的网络密钥

![](image/de3ea1a6fff8cc2fb4fa3ffd8f742dc3.png)

放好密钥后，就会明文显示

![](image/22adb2f241f61b543c3eb2d45df879d3.png)
