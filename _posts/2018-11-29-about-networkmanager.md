---
layout: post
title: NetworkManager 和 init scripts
categories: 
  - tools
categories:
  - linux
  - networking
date: 2018-11-29
---

在「一切皆文件」的指导思想下，Linux 中的所有硬件都被视作一个`/dev`下的设备文件，而用户态对设备的访问操作，就通过「设备文件」来进行。网卡设备当然也没有例外，在`/dev`路径下，可以看到类似`eth0`、`eth1`的网卡设备文件，或者`enp5s0`，如果系统启用了`systemd`。

在设备比较单一，配置过程简单的早期，Linux 的网卡配置通过一套所谓的`SysV init scripts`来完成。这些脚本调用`ifconfig`（或者更新的`ip`）工具，将系统预设或者用户定义的静态配置在系统启动后，直接应用到内核的网卡设备，以实现所谓的「配置持久化」。一般来说，网卡的配置，`DHCP` 还是静态地址，设备`MAC`地址，设备名称之类的信息，以一定的格式存在于系统文件中，`Debian/Ubuntu` 中位于 `/etc/network/interfaces`，`Fedora/RedHat` 则是 `/etc/sysconfig/network-scripts/ifcfg-*`。而`DNS`服务器配置在`/etc/resolv.conf`文件中。

随着硬件越来越小型化，计算机从庞然大物变成了桌面设备，人手多台。随之而来的就是各种热插拔，移动`WiFi`等等在系统启动后频繁变化的输入（再见，美好的静态田园时代 ;-)），系统启动阶段一次性初始化之后不闻不问越来越不合时宜，不能适应桌面`PC`的时代。
`NetworkManager`作为一个集中式的网络管理者应运而生。作为一个动态网络配置程序，`NetworkManager`有明显的几点优势：

* 保证 Linux 的网络管理足够简单，`It just works`。如果它发现系统没有配置网络但有可用的网卡，会自动生成一条配置，启用此网卡，保证系统可达。
* `NetworkManager`额外提供多种途径，命令行和图形界面，保证用户可以很容易的修改调整网卡设置。
* 命令行配置过无线网卡的人，会发现`NetworkManager`提供了接近`Windows/macOS`的界面体验。

`NetworkManager`作为后来者，要获得用户基础，其中一件重要的事情就是支持已有的`SysV init`配置。这种支持在`NetworkManager`里通过插件（见配置文件[`/etc/NetworkManager/NetworkManager.conf`](https://developer.gnome.org/NetworkManager/stable/NetworkManager.conf.html)）完成，在`Debian/Ubuntu`中默认启用的是`ifupdown`插件，`Fedora/RedHat`是`ifcfg-rh`插件，这些插件负责读写对应操作系统中已有的网络配置文件。

`ifupdown`插件的功能局限于读取配置，不支持写入。所以在`Debian/Ubuntu`中，通过还会启用另一个插件[`keyfile`](https://developer.gnome.org/NetworkManager/stable/nm-settings-keyfile.html)以支持配置写入。

而`ifcfg-rh`作为`Fedora/RedHat`系统的默认插件，负责读取和写入网络配置文件`/etc/sysconfig/network-scripts/ifcfg-*`。`ifcfg-rh`插件提供了详细的日志信息 

```bash
grep 'ifcfg-rh' /var/log/messages
```

可以看到各种各样的操作日志，包括设备是否启用、启用成功失败，失败原因等等。

```log
NetworkManager:    ifcfg-rh: parsing /etc/sysconfig/network-scripts/ifcfg-Auto_eth0 ...
NetworkManager:    ifcfg-rh:     read connection 'Auto eth0'
NetworkManager:    ifcfg-rh: parsing /etc/sysconfig/network-scripts/ifcfg-Test_LEAP ...
NetworkManager:    ifcfg-rh:     error: Missing LEAP identity
NetworkManager:    ifcfg-rh: parsing /etc/sysconfig/network-scripts/ifcfg-Test_Wifi_LEAP ...
NetworkManager:    ifcfg-rh:     read connection 'Test Wifi LEAP'
NetworkManager:    ifcfg-rh: parsing /etc/sysconfig/network-scripts/ifcfg-wlan0 ...
NetworkManager:    ifcfg-rh:     read connection 'System wlan0'
```

在引入`NetworkManager`之后，原本的网络配置文件`/etc/sysconfig/network-scripts/ifcfg-*`就变成`ifcfg-rh`插件的配置文件，`init scripts`的注意事项，例如`#`可以用作注释，带空格的字符串必须加引号，特殊字符要转义等等必须维持原样（因为还要支持`init scripts`操作）。

除此之外，还增加更多新的配置，例如`UUID`、`NM_CONTROLLED`，原有的一些配置项在`NetworkManager`下也有了更多的[取值和意义](https://developer.gnome.org/NetworkManager/stable/nm-settings-ifcfg-rh.html#differences_against_initscripts)。

| 字段| 意义    |  注意事项 |
| :--------- | :-------- | :----- |
| NM_CONTROLLED| 网卡是否由`NetworkManager`管理  | 默认`yes` |
| UUID| 设备唯一标识符  | 必须系统唯一，如未设置`NetworkManager`会自动生成 |
| PEERDNS|  PEERDNS=no 现在表示的是「不要把自动获取的 DNS 服务器加入 resolv.conf」 | |
| ONBOOT|  ONBOOT=yes 除了表示系统启动时启用设备，还表示任何时候可以自动连接 | |
| DNS1,DNS2,DNS3...|  DNS 服务器 | 可支持多个，但系统只使用前三个 |
| ...|   |  |
| ...|   |  | 

<br/>
`NetworkManager`提供命令行配置工具`nmcli`，也有带图形界面的`nmtui`/`nm-connection-editor`/`control-center`。虽然`NetworkManager`不鼓励直接操作`ifcfg-*`文件，但在大多数情况下，直接写文件还是更简单直接。如果修改了配置文件，别忘记通知`NetworkManager`，运行

```bash
nmcli con reload
```

重新加载所有连接的配置，或者

```bash
nmcli con load /etc/sysconfig/network-scripts/ifcfg-ethx
```

单独应用某一个文件。

由于`NetworkManager`的灵活性，在`NetworkManager`和`init script`共存的系统中，可能出现一些让人不太好理解的现象，比如设备名称与预期不一致，网卡无法启用。如果修改网络配置后出现非预期的情况，`NetworkManager`的日志信息大部分时候是个好的诊断入口。有问题，记得

```bash
grep 'NetworkManager' /var/log/messages
```
