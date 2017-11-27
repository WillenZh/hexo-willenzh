title: 用Hexo建立Github Page——你可能会遇到的问题
categories:
  - 技术相关
  - ''
tags:
  - CS相关
date: 2014-11-08 00:34:00
---
![](http://jayveestorage.qiniudn.com/QQ%E6%88%AA%E5%9B%BE20141107155113.png)
昨天研究了一下[Hexo](http://hexo.io/)在github上搭建个人博客的方法，参照了[Hexo在github上构建免费的Web应用](http://blog.fens.me/hexo-blog-github/)和[hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/)这两篇教程，终于算是对这个框架有了些许了解，感谢作者们的分享精神！教程写得比较详细，就不再赘述了。下面趁热打铁，把我遇到的问题总结一下。<!--more-->


### 目录：
1. 如何构建最简单的Github Page？
2. Hexo使用中的细节问题
3. Hexo的主题修改
4. 图床相关
5. Markdown相关


---

# **如何构建最简单的Github Page？**
按照我的理解，[Github Page](https://pages.github.com/)分为两类：一是为*个人*或*组织(Organization)*建立page，二是为*项目(repo)*建立page。
为个人或组织建立page，只要建立一个名字为*[name]*.github.io的库，并在`master`分支中添加index.html或index.md即可。需要注意的是，此处的*[name]*为个人或组织的名称，应该是**区分大小写**的，例如：我的github账号名为Jayveehe，则建立一个名为JayveeHe.github.io的库（它的详细地址为JayveeHe/JayveeHe.github.io）。第一次创建10分钟左右即可访问http://jayveehe.github.io/ 查看index的内容，太急的话会出现404，这应该是github服务器给page做DNS时应有的一定延迟吧。
对于建立项目的page，则在该项目下创建一个`gh-pages`分支，添加index.html或index.md即可，访问的地址则变为:**[name]**.github.io/**[projecName]**
所以通俗地说，Github Page的原理就是，系统会检查你特定名称的库或分支，给每个用户分配一个地址用于访问github page，如果有index.html或.md文件则显示其内容。


# **Hexo使用中的细节问题**
了解了Github Page的基本原理后，为了做漂亮的博客，我们需要找到一个能够帮助我们生成静态页面的框架，官方推荐使用[Jekyll](http://jekyllrb.com/)，而我们使用[Hexo](http://hexo.io/)。
安装和使用Hexo的教程参考上面提到的两个博客即可，我说说遇到的问题：
### **1. 问题**：在命令行输入`hexo d`会出现
```
Error: spawn ENOENT
    at errnoException (child_process.js:980:11)
    at Process.ChildProcess._handle.onexit (child_process.js:771:34)
```
>**解决**：我的原因是当初安装git的时候没有选择将git命令嵌入`cmd`中，而是单独使用`git bash`，导致在命令行中使用hexo自带的部署功能会出现无法调用git的情况。解决方法就是在`git bash`中使用`hexo`的命令。

---

### **2. 问题**：`hexo g`或`hexo s`的时候出现错误，无法生成静态页面
```
[error] { name: 'HexoError',
  reason: 'can not read a block mapping entry; a multiline key may not be an imp
licit key',
```
>**解决**：再次检查根目录`_config.yml`里面，各项参数的冒号后面一定要先加一个空格再填值。还有问题的话就具体分析了……

---

### **3. 问题**：`hexo s`或者`hexo d -g`发布网站后，访问出现![](http://jayveestorage.qiniudn.com/图片外链36a113f975708ddb6d87c8001488a8e0_b.jpg)
>**解决**：某些版本的Hexo（比如我用的2.8.3版本）好像没有添加`hexo-renderer-ejs` `hexo-renderer-marked` `hexo-renderer-stylus` 这几个插件，导致渲染有问题。到博客根目录运行`npm install XXX(插件名) --save` 安装完毕后应该就OK了，感谢知乎网友[waterox](http://www.zhihu.com/people/buffalo)的[解答](http://www.zhihu.com/question/23999043/answer/26581991)。

---

### **4. 问题**：如何在文章中设置*阅读更多*的断点？
>**解决**：编辑文章时，在想要设置断点的地方输入`<!--more-->`即可，想要修改*阅读更多*这个文字的话，就在相应主题文件夹里的`_config.yml`文件中修改`excerpt_link: more` 这个属性值即可。

---

# **Hexo的主题修改**
Hexo根据一定的脚本（或者说模块）进行静态网站生成，[主题](https://github.com/hexojs/hexo/wiki/Themes)(Themes)就是这些脚本的集合，想要自定义自己的博客布局，需要一些前端的知识。对于每个主题的布局，修改`layout`文件夹下的文件即可，具体的主题需要具体分析。值得一提的是，在Hexo中进行`new`操作时，里面有个`[layout]`参数，这里可以不填（默认为post），也可以填入`layout`文件夹下相应的`.ejs`文件名，这样会按照里面设定的模板进行文件创建了。
如果想要更全面的定制自己的主题，可以看看[这篇](http://blog.gfdsa.net/2013/04/09/hexo/hexolessontwo/)，同时学习一下默认的模板语言 [EJS](https://github.com/tj/ejs)和[Stylus](http://learnboost.github.io/stylus/)，接下来就尽情发挥你的想象吧，祝你成功。


# **图床相关**
在博客中插入图片需要填url，一般我们可以在博客根目录下的`source`文件夹中放入所需图片，直接填相对路径就可以了。但是这样文件是存储在Github服务器上的，国内访问有可能不是特别快，我们可以利用以下方法：

1. 使用新浪微博的相册功能，新建一个外链专用相册（当然你可以把它设为私密的），上传想要使用的图片到相册中，查看大图获取URL。这个方法的访问速度不错，流量限制应该比较宽松，但是可能操作有些麻烦。
2. 很多地方推荐的[七牛云存储](http://www.qiniu.com/)，注册用户每月免费1G流量，如果进行用户认证可以免费获取10G/月的流量，并且可以很方便地进行图片处理等操作，本人目前在用。


# **Markdown相关**
给几个不错的Markdown语法教程：

1. [献给写作者的 Markdown 新手指南](http://www.jianshu.com/p/q81RER)
2. [Markdown语法纪要](http://www.jianshu.com/p/0752bd0418df)
3. [Markdown 语法说明 (简体中文版)](http://wowubuntu.com/markdown/index.html#backslash)

在线的Markdown编辑器：

1. [Cmd Markdown](https://www.zybuluo.com) 本文就是用它完成的，无需注册即可使用，注册后可以保存文稿，推荐。
2. [简书](http://www.jianshu.com/) 似乎要注册才可以使用

---

就先这样吧，希望能给和我一样的新手们带来些许帮助，欢迎在本文评论区下进行讨论。
P.S.：正在研究主题修改，各位大神求带啊！


转载请注明出处（~~谜之音：滚滚滚谁会转你的破文啊~~），谢谢！