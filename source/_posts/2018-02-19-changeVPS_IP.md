---
layout: post
category: 杂记
date: 2018-02-19
title: 免费更换 bandwagon VPS IP 的方法
---

　　bandwagon 官方提供更换 IP 的[服务](https://bwh1.net/ipchange.php), 以前收费是 8 美元(折合人民币 50 元左右)，不知从什么时候降价到了 2.82 美元(折合人民币 18 元左右)，但其实在自己购买的 VPS 的 KiwVM control panel 里就可以更换 ip, 如果只是需要更换 ip 的话(~~比如自己搭的的梯子被禁这种事情~~)，可以通过这种方式免费更换ip.

　　首先进入 VPS 的 control panel 里，停止 VPS 的运行.

![stopVPSbtn](/downloads/stopVPSbtn.png)

　　然后选择左侧 Migrate to another DC.

![select_other_DC](/downloads/select_other_DC.png)

　　在以下列表中选择除后面为 current 以外的地区，如果只是切换 Los Angeles 的不同机房，速度上的差异应该不大，切换到其他地方速度不能保证.

![otherDC](/downloads/otherDC.png)

　　然后点击 confirm on next step, 切换到新机房用时几分钟到几小时不等，与 VPS 上用户的数据量有关，提示说会把全部的数据拷贝过去，切换过去后需要手动开启 VPS 和 shadowsocksR server 服务．

<!--
>　　后来许多人问我一个人夜晚踟蹰路上的心情，我想起的不是孤单和路长，而是波澜壮阔的海和天空中闪耀的星光．
-->