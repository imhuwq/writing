---
title: Win 10 无法发现局域网电脑的解决办法
date: 2019-03-30 21:22:50
categories:
- Tips
tags:
- win 10
---

最近整了一台 Surface Go，想在主力 PC 间实现远程桌面和文件共享。  
本以为都 2019 年了这些事情肯定“只需要点一下”就好了，实际上还是有很多坑。这篇文章就记录了 “Win 10 不能发现/连接局域网电脑” 的解决办法，基本上能解决所有网络上提到的 Win 10 联机问题(前提是在局域网)。  

- 确保在 “**控制面板\网络和 Internet\网络和共享中心\高级共享设置**” 勾选了 “启用网络发现” 和 “启用文件和打印机共享”，这是实现任何远程互动的前提
- 如果要使用远程桌面，必须允许远程访问计算机，让 Cortana 找出来“**必须允许远程访问计算机**”
- 如果要使用文件共享，必须在 “启用或关闭Windows功能” 中启用 **SMB**，让 Cortana 找出来“**启用或关闭Windows功能**”
- 让 Cortana 找出“**服务**”，设置以下服务自动开启：
  - PNRP Machine Name Publication Service
  - TCP/IP NetBIOS Helper
  - Computer Browser（Browser）
  - Function Discovery host Provider（FDPHOST）
  - Function Discovery Resouce  Publication(FDResPub）
  - Network Connections（NetMan）
  - Upnp Device Host（UpnpHost）
  - Peer Name Resolution Protocol (PNRPSvc)
  - Peer Networking Grouping (P2PSvc)
  - Peer Networking Identity Manager (P2PIMSvc)
- 最后为了省心，干脆**重启**个电脑吧
<!--more-->

顺带一提，如果发现局域网对拷异常的慢，不要惊慌，有两种可能:
- 无线对拷就是慢，因为天线是给各个设备轮流使用的
- 检查 “控制面板\网络和 Internet\网络连接” 里面网卡属性，看看当前协商的接口速度是多少，是否和网卡以及路由器的匹配。在这个年代基本不会有百兆以下的网卡和路由器，如果低于理论值，检查路由器设置 
- 信号干扰、数据包解析、加密解密

最后再提示一下，如果 Win 10 的局域网文件共享功能正常，但是 Steam 的家庭串流模式不能用(发现不了其它电脑)，试着禁用虚拟网卡，说不定有惊喜。  
