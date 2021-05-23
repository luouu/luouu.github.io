---
layout: post
title: wifi调试工具
categories: [wifi]
description: 
keywords: 
topmost: false
---

* TOC
{:toc}

wext为linux-2.6.18版本之前实现方式；用户空间使用ioctl方式访问驱动，设置无线参数或者获取无线参数，配置无线驱动进行联网操作；无线驱动事件到应用层的传递采用的netlink socket技术。

linux-2.6.18以后wifi驱动实现方式增加了nl80211，无论是用户层访问驱动还是驱动事件通知应用层、都采用的netlink技术。

用`lsusb`命令查看USB ID号，无线网上的硬件id为：0bda:xxxx。

## wpa_supplicant

支持OPEN、WEP、WPA(TKIP)、WPA2(AES)无线加密方式。下载地址：https://w1.fi/releases/
wpa_supplicant-0.8以前的版本，编译前对.config文件进行裁剪，去掉一些功能。

```conf
#CONFIG_DRIVER_HOSTAP=y
#CONFIG_DRIVER_ATMEL=y
#CONFIG_DRIVER_NL80211=y
#CONFIG_DRIVER_WIRED=y
CONFIG_WPS=y
CONFIG_TLS=internal
CONFIG_INTERNAL_LIBTOMMATH=y
```

```shell
cp defconfig .config
make CC=arm-himix100-linux-gcc clean
make CC=arm-himix100-linux-gcc -j4
arm-himix100-linux-strip wpa_supplicant
```

依赖libnl，openssl，dbus库，Makefie修改：

```makefile
CFLAGS += -I/home/luo_u/usr/hisi/dbus-1.12.10/include/dbus-1.0
LIBS += -L/home/luo_u/usr/hisi/dbus-1.12.10/lib -ldbus-1
CFLAGS += -I/home/luo_u/usr/hisi/expat-2.2.9/include/
LIBS += -L/home/luo_u/usr/hisi/expat-2.2.9/lib 
CFLAGS += -I/home/luo_u/usr/hisi/openssl-1.1.1/include
LIBS += -L/home/luo_u/usr/hisi/openssl-1.1.1/lib 
CFLAGS += -I/home/luo_u/usr/hisi/libnl-3.4.0/include/libnl3
LIBS += -L/home/luo_u/usr/hisi/libnl-3.4.0/lib 
```

http://www.linuxfromscratch.org/blfs/view/svn/basicnet/wpa_supplicant.html

```json
ctrl_interface=/var/run/wpa_supplicant/
update_config=1

network={
    ssid="SECULINK  R&D"
    scan_ssid=1
    psk="secueye1"
    key_mgmt=WPA-PSK WPA-EAP IEEE8021X NONE
}

#连接无密码的ssid字段，需要添加key_mgmt=NONE去连接这个网络
network={
        key_mgmt=NONE
        ssid="wifi-name"
}
```

```shell
./wpa_supplicant -i wlan0 -D wext -c wpa_supplicant.conf -B -dd
```

加`-dd`选项，显示调试打印。

* **错误**

```cmd
rfkill: Cannot open RFKILL control device
```

打开rfkill失败，这是一个控制接口应该在/dev/rfkill，看了下板子上，确实没有这个设备接口。打开内核配置`CONFIG_RFKILL`

## wpa_cli

```shell
wpa_cli -i wlan0 scan 　       #搜索wifi热点
wpa_cli -i wlan0 scan_result 　#显示搜索wifi热点
wpa_cli -i wlan0 status      　#当前WPA/EAPOL/EAP通讯状态
wpa_cli -i wlan0 ping        　#ping wpa_supplicant

wpa_cli -i wlan0 add_network   #添加一个网络连接,会返回<network id>
wpa_cli set_network <network id> ssid '"name"'  #ssid名称
wpa_cli set_network <network id> psk '“psk”'    #密码
wpa_cli set_network <network id> scan_ssid 1
wpa_cli set_network <network id> priority 1     #优先级

#连接指定的ssid
wpa_cli -i wlan0 select_network <network id>   

#连接无密码热点
wpa_cli set_network <network id> ssid "wifi-name"
wpa_cli set_network <network id> key_mgmt NONE
       
#断开连接
wpa_cli -i wlan0 disable_network <network id>   

#信息保存到默认的配置文件中
wpa_cli -i wlan0 save_config 

#列举保存过得连接
wpa_cli -i wlan0 list_network     

#使能制定的ssid
wpa_cli -i wlan0 enable_network <network id>   
```

## iwconfig

Wireles stools用来设置支持LinuxWireless Extension的无线设备。Wireles stools for Linux 和Linux Wireless Extension 由 Jean Tourrilhes在维护，由HP惠普赞助。

WirelessExtension (WE)是一组通用的API，能在用户空间对通用Wireless LANs进行配置和统计。它的好处在于仅通过一组单一的工具就能对各种各样的WirelessLANs进行管理，不管它们是什么类型，只要其驱动支持WirelessExtension就行；另一个好处就是不用重启驱动或Linux就能改变这些参数。

WirelessTools (WT)就是用来操作Wireless Extensions的工具集，它们使用字符界面，支持所有WirelessExtension。WirelessTools包括以下工具：

-　iwconfig：设置基本无线参数
-　iwlist：扫描、列出频率，比特率，密钥等
-　iwspy：获取每个节点链接的质量
-　iwpriv：操作WirelessExtensions 特定驱动
-　ifrename： 基于各种静态标准命名接口

下载地址：https://www.hpl.hp.com/personal/Jean_Tourrilhes/Linux/Tools.html
修改Makefile，编译。

```makefile
CC = arm-himix100-linux-gcc
AR = arm-himix100-linux-ar
RANLIB = arm-himix100-linux-anlib
BUILD_STATIC = y
```

只适用于OPEN和WEP加密方式。建议用iw工具替代。

```shell
iwlist wlan0 scan   #搜索ap
iwconfig wlan0 key <psk>  #验证密码
iwconfig wlan0 key open   #密码验证功能打开
iwconfig wlan0 essid <ssid>  #连接ap
```

在设置监听模式之前需要将无线网卡down掉，开启监听模式再up，然后在锁定信道.。

## iw

iw 是一种新的基于 nl80211 的用于无线设备的CLI配置实用程序。只能连接 OPEN和WEP 加密方式的热点。下载地址：https://mirrors.edge.kernel.org/pub/software/network/iw/

修改Makefile，依赖libnl库。

```makefile
CC = arm-anykav200-linux-uclibcgnueabi-gcc
CFLAGS += -I/home/luoyou/home/usr/anyka/libnl-3.4.0/include/libnl3
LIBS += -L/home/luoyou/home/usr/anyka/libnl-3.4.0/lib
```

```shell
export PKG_CONFIG_PATH=/home/luoyou/home/usr/anyka/libnl-3.4.0/lib/pkgconfig
make
```

```shell
iw help    # 帮助
iw list    # 获得所有设备的功能，如带宽信息和802.11n的信息
iw dev wlan0 scan   # 扫描
iw event            # 监听事件 
iw dev wlan0 link   # 获得链路状态 

iw wlan0 connect <essid>       # 连接开放的AP
iw wlan0 connect <essid> 2432  # 连接到2432频道的AP 

# 连接WEP的AP，d:default, 0:第0个密码, 1:第0个密码
iw wlan0 connect foo keys 0:abcde d:1:123456    

iw dev wlan1 station dump    # 获取station 的统计信息
iw dev wlan1 station get     # 获得station对应的peer统计信息
iw wlan0 set bitrates legacy-2.4 12 18 24    # 修改传输比特率 
iw dev wlan0 set bitrates mcs-5 4    # 修改tx HT MCS的比特率 
iw dev wlan0 set bitrates mcs-2.4 10  
iw dev wlan0 set bitrates mcs-5  # 清除所有tx比特率和设置的东西来恢复正常
iw dev　set txpower  []   #设置传输功率
iw phy　set txpower  []   #设置传输功率
iw dev wlan0 set power_save on  #设置省电模式
iw dev wlan0 get power_save     #查询当前的节电设定
iw phy phy0 interface add moni0 type monitor  #添加一个monitor接口
```
