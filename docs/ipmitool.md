ipmitool
=========
好像是一种管理工具。可以连接到服务器的串口输出
```
ipmitool -I lanplus -H 192.168.2.151 -U Administrator -P Adminpasscode sol activate
```
192.168.2.151 是IP地址  
Administrator 是用户名  
Adminpasscode 是密码  

当发现ipmi无法使用时，可以另起session，kill掉现在的连接
```
ipmitool -I lanplus -H 192.168.2.151 -U Administrator -P Adminpasscode sol deactivate
```
如果ipmitool 连接到了目标单板但是没有输出。有两种设置方法：

方法一：修改BIOS设置
```
BIOS -> Device Manager -> Console Preference Selection -> Preferred console Serial
```
方法二：修改OS的/etc/default/grub，在quiet后面添加console=ttyAMA0,115200
```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgg
b quiet console=ttyAMA0,115200"
GRUB_DISABLE_RECOVERY="true"
```
更新grub.cfg文件：
```
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```
请注意不同的OS有不同的更新方式，请参考[[grub]](grub.md)

以下所有命令需要先执行：
```
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode
```

#电源管理：
```CS
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode power off   	#(硬关机，直接切断电源)
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode power soft  	#(软关机，即如同轻按一下开机按钮)
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode power on    	#(硬开机)
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode power reset 	#(硬重启,断电上电)
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode power status	#(获取当前电源状态)
ipmitool -H 192.168.1.59 -I lanplus -U Administrator -P Adminpasscode chassis power cycle #（断电1秒后上电）
```

# 远程引导（档次有效）
```
chassis bootdev pxe 	#网络引导
chassis bootdev disk 	#硬盘引导
chassis bootdev cdrom 	#光驱引导
chassis bootdev bios  	#重启后停在BIOS 菜单 在1620没有问题
chassis bootdev pxe　	#重启后从PXE启动
```
# 读取系统状态
```
sensor list   #显示系统所有传感器列表
fru list　　　#显示系统所有现场可替代器件的列表
sdr list　　　#显示系统所有SDRRepository设备列表　
pef list      #显示系统平台时间过滤的列表
```

#系统日志类
```
sel elist　　　 #显示所有系统事件日志
sel clear　　　 #删除所有系统时间日志
sel delete ID   #删除第ID条SEL
sel time get    #显示当前BMC的时间
sel time set	#设置当前BMC的时间
```

#系统相关的命令
```
mc info       		#显示BMC版本信息
bmc reset cold      #BMC热启动
bmc reset warm   	#BMC冷启动
```

#网络接口相关命令
```
lan print 1								#显示channel1的网络配置信息
lan set 1 ipaddr 10.32.2.2    			#设置channel1的IP地址
lan set 1 netmask 255.255.0.0   		#设置channel1的netmask
lan set 4 defgw ipaddr255.255.0.254     #设置channel4的网关
lan set 2 defgw macaddr  <macaddr>　   #设置channel2的网关mac address
lan set 2 ipsrc dhcp         			#设置channel2的ip 源在DHCP
lan set 3 ipsrc static        			#设置channel2的ip是静态获得的
```
#通道相关命令
```
channel info　#显示系统默认channel
channel authcap channel-number privilege 　#修改通道的优先级别
channel getaccess channel-number user-id　#读取用户在通道上的权限
channel setacccess channel-number  user-id callin=on ipmi=on link=onprivilege=5   #设置用户在通道上的权限
```

#看门狗相关命令
```
mc watchdog get　#读取当前看门狗的设置
watchdog  off   #关掉看门狗
watchdog reset 　#在最近设置的计数器的基础上重启看门狗
```
#用户管理相关命令
```
ipmitool user list chan-id   				#显示某通道上的所有用户
ipmitool set password <user id>[<password>] #修改某用户的密码
ipmitool disable      <user id>　　			#禁止掉某用户
ipmitool enable       <user id>　　			#使能某用户
ipmitool priv         <user id> <privilegelevel> [<channel number>]　#修改某用户在某通道上的权限
ipmitool test         <user id> <16|20>[<password]>　#测试用户
```

# Board_connect

其他命令请参考：
```
https://blog.51cto.com/bovin/2128475
```