title: IPV6下利用GoAgent回避校园网收费
tags:
  - CS相关
categories:
  - 技术相关
date: 2015-01-04 20:34:00
---
对于没钱或者懒得买VPN的人来说，GoAgent算是一个低成本的科学上网~~（fanqiang）~~方案了，速度可满足普通的网页浏览需求，并且只需要一个gmail账号即可使用。

在大部分校园内，流量收费通常只计算IPV4，而IPV6则暂时具有免费、带宽大等特点。GoAgent，确切地说Google App Engine（GAE）是支持IPV6访问的，这就让使用GoAgent回避校园网流量收费成为可能。利用GAE的服务器进行IPV4网站访问，再通过IPV6取回本地浏览，即可以减少计费流量。
相比修改DNS且只用IPV6的方案，本方案更加灵活。

下面简单介绍一下具体的使用方法和一些小技巧。<!--more-->
**已有GoAgent的可以跳过基础篇**

---

## 基础篇：GoAgent的获取与使用
1. 访问[goagent的github主页](https://github.com/goagent/goagent)下载最新版本的goagent（下方有地址，或者直接点右边栏的Download ZIP）
2. 到[Google App Engine](https://appengine.google.com/)下使用gmail账号登陆，这一步可能需要“科学上网”，这个鸡生蛋还是蛋生鸡的问题各位自行解决……
![gae](http://jayveestorage.qiniudn.com/图片外链QQ截图20150104184527.png)
新建一个应用程序，其中**Application Identifier**(appid)一定要记住，后面会用到。
3. 将第一步下载的文件解压，得到
![](http://jayveestorage.qiniudn.com/图片外链搜狗截图15年01月04日1830_1.png)
进入`server`文件夹，打开`uploader.bat`，依次输入第二步设置的APPID、gmail邮箱名、密码，等待上传完毕即可。
4. 进入`local`文件夹，打开`proxy.ini`，如图
![](http://jayveestorage.qiniudn.com/图片外链proxyini.png)
在APPID处填入你的*appid*，并修改**ipv6=1**
5. 打开`local`下的`goagent.exe`，右键状态栏里的图标，如图
![](http://jayveestorage.qiniudn.com/图片外链ieproxy.png)
设置IE代理并选择127.0.0.1:8087。
6. 在常用的浏览器的代理服务器设置中（比如此处的搜狗浏览器），选择`使用IE的代理设置`。
![](http://jayveestorage.qiniudn.com/图片外链sougou.png)

**至此，goagent就可以正常使用了。**


---
## 技巧篇：自动切换代理
所以使用IPV6版GoAgent的关键点就在于修改`proxy.ini`里的`ipv6=1`。
但是上面设置的IE代理属于一刀切的方法，你在浏览器中输入的所有网址，都将经过GAE的服务器进行一个转发。换句话说，在所有网站的眼中，你的身份就是一个在美国google机房上网的用户。
现在问题来了，作为一个学生还经常需要查询内网信息（教务处、选课系统等），应该怎么办？
最简单的方法，就是在需要查询内网信息时手动禁用代理，但是这样很麻烦。


### Chrome的代理自动切换
有经验的同学应该用过chrome的SwitchySharp插件，它会根据一定的规则来指定某个网页使用什么代理设置，如此便可无缝地浏览内外网。今天发现SwitchySharp升级为SwitchyOmega了，感觉不错。

下面介绍如何使用Chrome的*SwitchyOmega插件*进行代理的自动切换：
1. chrome应用商店搜索**switchyomega**并安装，之后在地址栏右边会出现它的图标，点击进入选项。
2. 在选项中的左侧边栏选择`新建情景模式`，代理协议选择`HTTP`，服务器地址：`127.0.0.1`，端口：`8087`
![](http://jayveestorage.qiniudn.com/图片外链dailiqingjing.png)
3. 左侧边栏情景模式选择自带的`auto switch`，添加规则。此处北邮的同学可以按照我的设置
![](http://jayveestorage.qiniudn.com/图片外链auto.png)
或者也可以自己进入某个需要添加规则的网址下，点击地址栏右侧的SwitchyOmega图标，接下来点击`添加条件`
![](http://jayveestorage.qiniudn.com/图片外链step1.png)
然后进行相应的规则添加
![](http://jayveestorage.qiniudn.com/图片外链step2.png)。
4. 在浏览网页时使用`auto switch`模式即可
![](http://jayveestorage.qiniudn.com/图片外链confirm.png)

**如果按照我的设置，北邮的同学应该可以直接使用IPV6访问论坛、BT（这两个使用IPV6的代理也行）以及网关、信息门户、和教务系统了，同时也可以自行添加想要直接访问的网站。**


### 其他浏览器的代理自动切换
有些同学又说了，“谷粉滚粗啊，我是火狐党！”“我就爱360浏览器！”“除了搜狗都是异端！”“微软帝国万岁！”
那么其他浏览器如何进行自动切换呢？
火狐党的自己搜插件好了，其他的浏览器我们可以使用pac文件。
记得基础篇里面提到的“IE代理设置”么？通过使用pac脚本可以自动切换代理:
1. 首先导出自定义的pac文件。在chrome的SwitchyOmega选项页面中，选择auto switch，点击上方的`导出PAC`，放到任意位置。
2. 进入任意浏览器（或者系统设置）里的Internet选项-连接-局域网设置，如图设置：
![](http://jayveestorage.qiniudn.com/图片外链pac.png)
其中脚本地址改为自己的
3. 此后启动GoAgent就不用再设置IE代理了，只需在浏览器的代理服务器设置中选择`使用IE的代理设置`即可。**大功告成**。

---

## 一些说明
1. 使用时无需登陆网关。通过GoAgent代理的方式同样无法登陆QQ，可以使用webQQ或者登陆网关。北邮的同学可以直接访问10.3.8.211或者gwself.bupt.edu.cn进行网络账号管理，同时也可以通过代理访问其他网页。
2. 出于安全性考虑，使用网银、支付宝时最好直接连接。
3. 大部分可以设置代理服务器的软件都可以使用GoAgent避免计费流量，实测有[网易云音乐](http://www.baidu.com/link?url=h42YmJq020n66vobwNeihq_coHOpU1sMNI3JFM6xU8W)、[百度云盘](pan.baidu.com/)、[BiliLocal](https://github.com/AncientLysine/BiliLocal)等
4. 每个GAE app有一个1G/天的流量限制，但是一个账号可以申请多个app，具体的方法不再介绍。
5. 此方法观看网络直播时效果不佳，但是在线视频还是可以的，实测土豆、优酷等可用（当然包括AB站！）
6. IPV6用户也可以顺便使用改HOST的方法访问神秘网站，这里给一个[IPV6可用的HOST更新页面](http://serve.netsh.org/pub/ipv6-hosts/)
7. proxy.ini文件中的ip改为0.0.0.0后，可以让局域网内的用户使用本代理，这样在宿舍使用手机上网时也可通过电脑的goagent回避收费（自行设置手机的网络代理）。同样的，手机QQ和微信也不能在代理下登陆。



**最后我想说的是，即便很省着用，每个月10G的免费流量也根本不够嘛！！！**

*转载注明[Jayvee's Blog](http://jayveehe.github.io/2015/01/04/ipv6/)*