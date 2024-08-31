# 慧湖不通



> 文星宿舍宽带网络跨平台解决方案



## 路由器平台适配

| Platform                         | 设备数量解锁       | 路由器&子网解锁    | 自动登录     | 自动部署           | 集群部署     |
| -------------------------------- | ------------------ | ------------------ | ------------ | ------------------ | ------------ |
| OpenWRT                          | :white_check_mark: | :white_check_mark: | :x:          | :x:                | :x:          |
| Windows/Linux<br/>-VM方案        | :white_check_mark: | :white_check_mark: | 开发中       | :white_check_mark: | :x:          |
| Windows/Linux<br/>-Container方案 | 后续适配           | 后续适配           | 后续适配     | 后续适配           | 后续适配     |
| Container                        | 后续适配           | 后续适配           | 后续适配     | 后续适配           | 后续适配     |
| Android&Mac                      | 暂无适配计划       | 暂无适配计划       | 暂无适配计划 | 暂无适配计划       | 暂无适配计划 |
| Mac                              | 暂无适配计划       | 暂无适配计划       | 暂无适配计划 | 暂无适配计划       | 暂无适配计划 |



## 特殊硬件适配

| Name                                                 |                      |                   |
| ---------------------------------------------------- | -------------------- | ----------------- |
| NewWifi3<br>Unlook by Breed@RT-N56UB1-newifi3D2-512M | 已做固件与EEPROM适配 | 已测试(2024/8/31) |
| 待定                                                 |                      |                   |



## 原理&功能

几个常见检测策略的组合。除认证外，大部分功能部署在光猫侧。仅适用于文星，其余宿舍区未测试。



### 1.解锁连接设备数量

HHT通过MAC地址计算已连接设备，最多允许三台。

使用网线连接时，每个适配器均参与连接设备计算。

使用路由器时，光猫收到的以太网帧相关地址字段均为路由器的MAC地址，对外仅显示单个设备连接。

**将路由器的地址改为任意一台已经过认证的设备即可。**





### 2.解锁路由器&子网限制

TTL字段在经过每个路由器时会减1，HHT计算每个Packet的TTL字段来确定宿舍内是否使用了路由器，当收到的TTL!=设定值则重定向到blacklist.php。

宿舍使用路由器(非AP模式)后，部分packet的TTL字段在离开本机网络适配器后会-1，在经过路由器后再次-1，在光猫端检测时小于设定值，因此进行阻断。

CLASH等代理与游戏加速器软件会在部分系统中会创建虚拟适配器，经过代理的流量在系统内部已经对packet的TTL字段进行了修改,因此在正常连接的情况下也会被阻断(不是HHT进行了相关特征的检测)。

**在流量出口处将所有packet的TTL设为128(或较大的数)即可规避检测。注意不要设置TTL不变，因为内部网络可能有多个子网，例如使用加速器的情况下仍会被检测。**





### 3.自动登录

HHT使用微信二维码进行认证。

当需要上网的设备获取到上级设备发来的登录页面时，使用局域网将内容数据转发至内网的手机即可。手机后台使用推送服务将解码后的二维码(或直接发送链接)发送至你的微信。

**能够实现(已经配置过的)设备(在后续连接时)你的微信会自动收到认证用的二维码**，然后长按手动扫码

安卓端使用的方案是Termux+ServerChan([Server酱·Turbo版 | 一个请求通过API将消息推送到个人微信、企业微信、手机客户端和钉钉群、飞书群 (ftqq.com)](https://sct.ftqq.com/))

二维码解码方案是公开API，有root的设备可以尝试直接调用微信进行二维码扫描

```
用于认证url的示例结构:
https://broadband.215123.cn?clientId=6d6bc6f3b5f04107a5fc1c62e39dd5f4&uuid=0d619fe9fa9c4fe4beb4a7b50c78941d&type=pc
```





### 4.光猫破解

获取超管密码，解锁SSH等权限

**疑似已有成熟方案，此处不对该内容进行介绍，也不建议在宿舍区内使用，如需要请自行搜索。**

注: 可能使用了TR069进行在线配置下发，不要删除该连接 





### 5.单线多拨

绕过运行商带宽限制，请使用cat5e以上的网线

**此处不对该内容进行介绍，也不建议在宿舍区内使用。**





### 6.链路聚合

链路聚合可以进行局域网内的流量调度，吞吐量<=所有宽带带宽的总和

**需要单个宿舍内开通多个相同运行商的宽带&你室友的支持**

推荐使用connectify dispatch进行OS级别的聚合，无需对现有网络环境进行改造。





### 7.集群部署

以容器的方式部署在你和室友的电脑上。自动检测上线情况，自动投票一台当前可用的容器当作软路由，不需要每次带电脑回宿舍的时候开一边虚拟机。这个是在之前项目上改的



-----



## 方案



### PLAN A - 使用现有设备 - Route模式


![image-20240831205021408](https://github.com/ZTX1836255060/HuiHuBuTong/blob/main/assets/image-20240831205021408.png)

> 无需新设备，当需要对每台设备进行修改，路由器重启后需要重新认证

- 路由器保持路由模式，启用DHCP服务器等功能，光猫网线连接WAN口
- 路由器设置内将MAC地址修改为任意已经过认证的设备
- 修改每台需要联网的设备TTL的配置，
  - Windows在HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters处修改，其他平台类似
- 核心原理是packet抵达路由器时，其TTL字段在出路由器-1后刚好合法，以太网帧又被替换为已认证地址(路由器MAC)，此时HHT只能检测到单台设备。





### PLAN B - 使用现有设备 - 设备路由模式 - EASY MODE

![](https://github.com/ZTX1836255060/HuiHuBuTong/blob/main/assets/image-20240831211032773.png)

> 无需新设备，不使用虚拟机，最多仅需要修改一台电脑的TTL设置，但该电脑需要保持开启

- 一台经过认证的电脑进行连接光猫，并进行网络共享
  - 使用windows自带的无线网络共享: 设置->网络->移动热点
  - 或使用有线(有多余接口的情况下)连接开启AP模式的路由器，其余设备无线连接路由器，下游设备无需认证。





### PLAN C - 使用现有/专有设备 - 设备路由模式

![image-20240831213854395](https://github.com/ZTX1836255060/HuiHuBuTong/blob/main/assets/image-20240831213854395.png)

> 需要支持运行Container/VM的电脑作为路由器
>
> 或
>
> 使用现有软路由
>
> 需要有相关经验

- 软路由进行认证，TTL修改，MAC替换
- AP提供无线网络
- 根据设备类型按下表进行配置

| 设备                | 方案                                                         | 文件              |
| ------------------- | ------------------------------------------------------------ | ----------------- |
| 软路由              | 无直接适配，建议使用OpenWRT自行编译<br>或者使用虚拟机运行Ikuai镜像 | HHBT_OPENWRT.ZIP  |
| Windows10/11或Linux | VM方案,底层为配置好的Ikuai虚拟机<br>1.使用VMWare导入虚拟机镜像<br>2.设置适配器关联 | HHBT_VM_IKUAI.ZIP |
| Windows10/11或Linux | Container方案                                                | 暂无              |

Ikuai虚拟机是在之前项目的镜像上修改的，里面可能设置了自动重启，记得关掉





### PLAN D - 使用专用设备

![image-20240831213521544](https://github.com/ZTX1836255060/HuiHuBuTong/blob/main/assets/image-20240831213521544.png)

> 需要能够安装OpenWRT/相关系统的可刷机路由器

- 已完成对RT-N56UB1-newifi3D2-512M的完整适配，如果你有该设备可直接导入备份的固件，无需任何其他操作
  - 账号为root，密码为huihubutong
  - 建议使用Breed进行固件更新
  - MAC地址在Breed内修改(使用EEPROM-方案为公版，勿使用其他版本)
  - 文件为HHBT-NEWIFI3D2-BREED.ZIP
- 部分华硕系设备理论上可以使用OpenWRT，但未经过测试。
  - 如果你有时间参与适配，建议开源上传并分享至相关论坛





## 文件校验

| Name                     | DESC                            | MD5                              |
| ------------------------ | ------------------------------- | -------------------------------- |
| HHBT-NEWIFI3D2-BREED.ZIP | 固件与Breed控制台               | 4B40242143375209729C14430221A53B |
| HHBT_VM_IKUAI.ZIP        | ikuai OS,用于VMWare的虚拟机文件 | 80E0B9840AAC597B234141D48492912E |
| HHBT_OPENWRT.ZIP         | dump文件                        | FFA75B3B1DCB63E272456399ED336C79 |





## 声明

本项目提到的所有内容与XJTLU&/慧湖通无关

本项目开发时间远早于对应宿舍区硬件更新的时间，不针对任何实体与个人，仅用于学习。除自动二维码发送功能外，其余所有文件(包括但不仅限于HHBT-NEWIFI3D2-BREED.ZIP&HHBT_VM_IKUAI.ZIP& HHBT_OPENWRT.ZIP内的文件)与功能已于**2024年4月20日**前学习计算机网络时完成开发，本项目也不包含单线多拨或VPN等可能存在安全隐患的代码

本项目不盈利，其余见开源协议
