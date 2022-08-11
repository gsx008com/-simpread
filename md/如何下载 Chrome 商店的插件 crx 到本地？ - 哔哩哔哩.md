> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.bilibili.com](https://www.bilibili.com/read/cv6619562/)

> 由于不能经常爬墙，上谷歌，想把之前从 chrome 商店安装的插件保存成本地的 crx 文件用于以后离线安装。

由于不能经常爬墙，上谷歌，想把之前从 chrome 商店安装的插件保存成本地的 crx 文件用于以后离线安装。

查了几个方法：

1.  用浏览器的打包功能，打包出来的 crx 直接拖入会显示损坏。解包安装不能移动文件位置。
    
2.  插件 id 解析网站，查到的都不能用
    

最后找到了一个直接从 chrome 商店下载 crx，缺点是要爬墙。不过能保存 crx，并且直接拖入就能安装。

以下为转载：https://blog.csdn.net/pcdiyer/article/details/102506210

![](http://i0.hdslb.com/bfs/article/02db465212d3c374a43c60fa2625cc1caeaab796.png)

从 Chrome 商店搜索插件或直接通过链接进入插件的详情页面，其中地址栏最后的那一串字符就是 ID，复制下来。

https://chrome.google.com/webstore/detail/vuejs-devtools/ljjemllljcmogpfapbkkighbhhppjdbg

通过 ID 拼凑一下下面的地址直接访问就可以下载 crx 到本地。

https://clients2.google.com/service/update2/crx?response=redirect&os=win&arch=x64&os_arch=x86_64&nacl_arch=x86-64&prod=chromecrx&prodchannel=&prodversion=77.0.3865.90&lang=zh-CN&acceptformat=crx2,crx3&x=id%3Dljjemllljcmogpfapbkkighbhhppjdbg%26installsource%3Dondemand%26uc 

前提是你的电脑能打开谷歌，或者 **** 一下。

点击浏览器地址栏区域最右侧的菜单按钮，在菜单中选择更多工具，再点击扩展程序，或者直接在地址栏访问 chrome://extensions/ 进入扩展程序管理页面。

直接将下载好的 crx 插件文件拖到扩展程序页面就可以安装。

如果不行，请把开发者模式打开，也在扩展程序管理页面，然后将插件解压到一个空的英文目录下，通过加载已解压的扩展程序的功能去找到你解压的插件目录即可。

![](http://i0.hdslb.com/bfs/article/4adb9255ada5b97061e610b682b8636764fe50ed.png)

下载得的 crx 可直接拖入 chrome 扩展页安装！！