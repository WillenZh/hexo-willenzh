title: 微信公众平台接入初探
tags:
  - Linux
  - Python
categories:
  - 技术相关
date: 2015-01-26 17:50:00
---
最近出于兴趣，弄了个微信公众号玩玩，自从审核通过后就开始琢磨如何接入平台。趁着还记得，说说如何接入公众平台，并用[Nginx](http://nginx.org/)+[Flask](http://flask.pocoo.org/)搭建一个简单的用于自动回复消息的服务器后台。
<!--more-->
推荐阅读：[在Ubuntu上使用Nginx部署Flask应用](http://www.oschina.net/translate/serving-flask-with-nginx-on-ubuntu?cmp)

## 微信公众平台上的设置
首先是需要有一个公众平台的账号，到[微信公众平台](https://mp.weixin.qq.com)注册一个，提交身份信息后等待审核，只有审核通过以后才能进入开发者中心。

审核通过后，进入开发者中心，要求进行服务器验证。如下图：
![](http://mp.weixin.qq.com/wiki/static/assets/ce21f9e7d08b0f553032261b23c43b77.png)
其中，URL填写自己服务器的地址，Token为自己自定义的一个字符串，如果下方“消息加解密方式”选择了包含密文的选项（第二第三个），那么EncodingAESKey则需要填入一个自定义的加密密钥。
填好后，如果服务端也准备完毕，点击提交则微信方的服务器就会向所填URL发起验证用GET请求。

## 服务端需要做的事
简单来说服务器验证的过程是这样的：微信的服务器会向你预留的URL发送一个GET请求，其中包含了时间戳(`timestamp`)、随机数(`nonce`)、一个随机字符串(`echostr`)以及一个根据时间戳、随机数还有你自定义的token加密而成的签名字符串(`signature`)。
你所要做的就是根据自己定义的token+收到的时间戳+收到的随机数进行sha1加密，然后与发来的签名字符串进行对比，如果一样的话就说明该条信息是来自于微信服务器的，此时在response中原样返回收到的那个随机字符串echostr即可。
如果不考虑安全性且嫌麻烦的话，应该可以无视加密验证过程，直接返回echostr即可完成服务器验证工作。
具体的接入文档可以看[微信官方的文档](http://mp.weixin.qq.com/wiki/17/2d4265491f12608cd170a95559800f2d.html#)

## 搭建简易的消息服务器
使用阿里云VPS作为我们的服务器，由于不想再写Java的servlet之类的东西了，出于简捷和学习的目的，我选择python的flask框架进行消息的响应，同时为了更好的利用宝贵的80端口，采用Nginx进行反向代理。Flask和Nginx的基本安装及基础配置可以很容易的搜到，下面我说说在实践中遇到的问题和解决的关键。
- **服务器验证的响应程序**
稍微改写一下flask的helloworld程序即可完成验证工作:

```python
    #encoding:utf8
    __author__ = 'Jayvee'
    from flask import Flask, request, make_response
    import hashlib
    import MsgParser

    app = Flask(__name__)


    @app.route('/')
    def hello_world():
        return 'Hello World!'


    @app.route('/weixin', methods=['GET', 'POST'])
    def weixin():
        if request.method == 'GET':
            if len(request.args) > 3:
                temparr = []
                token = "你的token"
                signature = request.args["signature"]
                timestamp = request.args["timestamp"]
                nonce = request.args["nonce"]
                echostr = request.args["echostr"]
                temparr.append(token)
                temparr.append(timestamp)
                temparr.append(nonce)
                temparr.sort()
                newstr = "".join(temparr)
                sha1str = hashlib.sha1(newstr)
                temp = sha1str.hexdigest()
                if signature == temp:
                    return echostr
                else:
                    return "认证失败，不是微信服务器的请求！"
            else:
                return "你请求的方法是：" + request.method
        else:  # POST
            print "POST"
            xmldict = MsgParser.recv_msg(request.data)
            reply = MsgParser.submit_msg(xmldict)
            response = make_response(reply)
            response.content_type = 'application/xml'
            return response


    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=9527,debug=True)
```
其中，验证部分只在GET方法中进行，上述代码中的POST方法部分是后面进行消息响应的相关代码。又假设我们将代码保存为weixin.py，那么在服务器中使用`python weixin.py`就能启动服务器，此时访问`http://服务器外网IP:9527/weixin` 就会得到 “你请求的方法是：GET” 的返回。
但是这样没有用到80端口，是无法在微信公众平台上验证通过的。同时，我们的服务器上也有其他程序需要用到80的端口，而又不希望weixin.py独占，那么我们就需要Nginx进行反向代理。
- **Nginx的反向代理设置**
Nginx的方向代理可以将外网请求的某些URL路径转发到内网中的某个服务器上，假设我们服务器的IP为127.0.0.1，那么现在的需求就是当我们在访问`http://127.0.0.1/weixin`时,实际访问的是`http://127.0.0.1:9527/weixin`。Nginx环境的搭建此处不再赘述，那么如何将请求转发到9527端口的/weixin下呢？
在`etc/nginx`下编辑`nginx.conf`文件，修改默认的配置文件。于http项内增加一个server

```
server {
	listen 80;
	server_name 127.0.0.1;
	location /weixin {
		proxy_pass http://127.0.0.1:9527;   
	}
}
```
意思是将所有80端口下对/weixin地址进行访问时（即http://127.0.0.1/weixin） 时，将会转到http://127.0.0.1:9527/weixin
保存退出后，cd到/usr/sbin目录下使用

```
sudo nginx -s reload
```
进行Nginx服务器的重新启动，之后应该可以验证成功了。
- **简易的消息回复程序**
通过审核而又未进行认证的公众账号所能使用的接口十分有限，其中比较常用的有消息收发接口，它们的调用权限与频次如图：
![未认证接口](http://jayveestorage.qiniudn.com/图片外链搜狗截图15年01月27日0339_1.png)

下面简单说说如何实现接收用户发来的消息并进行回复的功能：
**1. 解析用户发送的消息**
接收普通消息的接口，官方文档也有[详细介绍](http://mp.weixin.qq.com/wiki/10/79502792eef98d6e0c6e1739da387346.html)，大致的过程是：每当用户向公众号发送消息，微信服务器就将这个消息编成一个xml文本，通过POST方法提交到之前我们预留的URL上，我们要做的就是解析这个xml文本并作出相应的回复。需要注意的是，**这个xml文本是在request.data里面而不是在表单里**。一个普通的文本消息xml格式如下：  

```
 <xml>
 <ToUserName><![CDATA[toUser]]></ToUserName>
 <FromUserName><![CDATA[fromUser]]></FromUserName> 
 <CreateTime>1348831860</CreateTime>
 <MsgType><![CDATA[text]]></MsgType>
 <Content><![CDATA[this is a test]]></Content>
 <MsgId>1234567890123456</MsgId>
 </xml>
```
![](http://jayveestorage.qiniudn.com/图片外链QQ截图20150127133941.png)
在接收消息时其实也可以做消息来源验证，大概的流程和上面的服务器验证一样。
一切从简的话，只需提取`ToUserName`、`FromUserName`、`Content`这三个内容（其他的参数可以在后期深入扩展时利用，在此不再赘述），针对Content的内容进行相应的操作。
**2. 向用户发送回复消息**
在上一节中，接收到消息后如果5秒内做出响应，就会自动回复给相应的用户，我们需要做的就是编制一条相同格式的xml文本返回给微信服务器。如果只做简单的回复，就只调换`ToUserName`、`FromUserName`两者的位置，并添加相应的`Content`即可。
详细的说明也可以看官方文档-[被动回复用户消息](http://mp.weixin.qq.com/wiki/14/89b871b5466b19b3efa4ada8e577d45e.html)
**3. 相关的python代码**
在上文中的服务器验证代码中，出现了POST方法的响应分支，里面的`MsgParser`是我自定义的一个文件。
python中解析xml文本的方法很多，这里我使用xml.etree.ElementTree提供的方法进行操作，相关代码如下：

```python
# coding=utf-8
__author__ = 'Jayvee'
import time
import xml.etree.ElementTree as ET
import sys

reload(sys)
sys.setdefaultencoding('utf8')

def recv_msg(oriData):
    """
    获取从微信服务器post而来的消息
    :param oriData: post的data
    :return:返回一个包含发送者、接收者、消息内容的字典
    """
    xmldata = ET.fromstring(oriData)
    # 获取发送方的ID
    fromusername = xmldata.find("FromUserName").text
    # 接收方的ID
    tousername = xmldata.find("ToUserName").text
    # 消息的内容
    content = xmldata.find("Content").text
    xmldict = {"FromUserName": fromusername,"ToUserName": tousername, "Content": content}
    return xmldict


def submit_msg(content_dict={"": ""}, type="text"):
    """
    编制回复信息
    :param content_dict:
    :param type:
    :return:
    """
    toname = content_dict["FromUserName"]
    fromname = content_dict["ToUserName"]
    content = content_dict["Content"]
    content = "对啊，%s，然后呢" % (content)
    reply = """
    <xml>
        <ToUserName><![CDATA[%s]]></ToUserName>
        <FromUserName><![CDATA[%s]]></FromUserName>
        <CreateTime>%s</CreateTime>
        <MsgType><![CDATA[text]]></MsgType>
        <Content><![CDATA[%s]]></Content>
        <FuncFlag>0</FuncFlag>
    </xml>
	"""
    resp_str = reply % (toname, fromname, int(time.time()), content)
    return resp_str
```
一切从简就没有进行异常处理之类的操作，目前为止就成功设置了一个简易的消息自动回复后台。

## 总结
本文中介绍了如何最简化地接入微信公众平台，由于是**初探**，所以没有做太多的深入工作。Nginx和flask的其他特性也没有使用，一般来说也推荐使用**Nginx+flask+uWSGI**的组合进行服务端的搭建。
目前只是实现了一个高冷的聊天机器人功能（如果这个也算聊天机器人的话。。。）相关的应用就要各位展开想象了。
本文到此为止，谢谢观看！


*转载注明[Jayvee's Blog](http://jayveehe.github.io/2015/01/26/nginx-flask/)*
