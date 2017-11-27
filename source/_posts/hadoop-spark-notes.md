title: Hadoop Streaming和Spark的一些坑
date: 2017-05-06 19:16:14
tags:
    - CS相关
    - 大数据
    - Python
categories:
    - 技术相关

    
---
## 前言：
在工作中遇到了一些Hadoop Streaming和Spark in Python相关的坑，记录于此，以供参考。
<!--more-->
## 一、Hadoop Streaming相关
首先，如有任何疑请问优先查询[官方文档](http://hadoop.apache.org/docs/r2.8.0/hadoop-streaming/HadoopStreaming.html#Streaming_Command_Options)
### 1. Streaming样例

```shell
hadoop fs -rmr hejiawei03/hadoop_usertrace_result;
hadoop jar /usr/local/hadoop/hadoop-2.7.1/share/hadoop/tools/lib/hadoop-streaming-2.7.1.jar \
-archives libs.tgz#pack \
-D mapred.map.tasks=5 \
-D mapred.reduce.tasks=5 \
-D mapred.job.name='process usertrace for test' \
-file mapper.py -mapper mapper.py \
-file reducer.py -reducer reducer.py \
-file libs.zip \
-input hejiawei03/usertrace \
-output hejiawei03/hadoop_usertrace_result;
```

#### **说明**：
1. 其中`-file`参数是将指定文件分发至集群各台服务器的相应临时工作路径上，在更新的版本中推荐使用`-files`参数，后面跟上用逗号分隔的多个文件名（或路径）
2. `-archive`参数用于指定相应的压缩文件，它们会被分发并解压至各slave服务器的临时工作目录下。

### 2. 关于`-archives`解压后在各节点服务器上的位置
与`-file`参数类似，`-archives`参数后的文件会被自动解压，并且在默认情况下会通过一个命名为<文件名>的软链接symlink指向解压后的文件夹。例如：`-archives libs.tgz`会将解压后的文件夹用链接`#libs.tgz`来表示。

假设libs.tgz解压后的文件夹结构如下：

![文件夹结构](http://7qnaj2.com1.z0.glb.clouddn.com/%E6%96%87%E4%BB%B6%E5%A4%B9%E7%BB%93%E6%9E%84.png?imageView2/2/w/150/h/150)

则其中`six.py`的相对路径可以用`libs.tgz/libs/six.py`来表示，以此类推。搞清楚这个路径十分重要。

## 二、Spark相关
使用spark-submit命令进行程序部署，该命令下同样有`--files`和`--archives`等参数，其用法和Hadoop Streaming的类似，不再赘述。

同样地，遇到任何问题首先看[官网](http://spark.apache.org)

### 1. 使用pyspark进行spark操作
简单的例子可见官方repo：[https://github.com/apache/spark/tree/master/examples/src/main/python](https://github.com/apache/spark/tree/master/examples/src/main/python)

这里有一个非常详细的[博客](http://colobu.com/2014/12/08/spark-programming-guide/)，翻译自官方的[Spark Programming Guide](https://spark.apache.org/docs/latest/programming-guide.html)，强烈推荐！
Spark中有很多算子，这是区别于Hadoop的一大特点，此外，编程及理解算子的关键是理解RDD中的键值对结构。

### 2.在使用各算子时，一定要确认传入的是\<k,v\>中的v还是整个\<k,v\>

## 三、使用Python进行程序部署遇到的坑
在使用python编写hadoop streaming job以及spark程序时，常常遇到集群机器上没有需要的第三方库，因此需要依靠`-archives`以及`-file`参数将第三方包分发至各节点机器。

在python脚本内，可以使用**路径添加**或者**[zipimport](https://docs.python.org/2/library/zipimport.html)**的方式进行包导入。

### 1. 路径添加
假设第三方库arrow的相关文件在当前目录下的`./libs/arrow`中，则可以通过下列方式进行import：

```Python
import os
import sys

PARENT_PATH = os.path.dirname(os.path.abspath(__file__))
print PARENT_PATH
sys.path.append(PARENT_PATH)
sys.path.append('%s/libs' % PARENT_PATH)

import arrow
```

### 2. zipimport
将第一节第2小节处的文件夹打包为`pkgs.zip`，则可以用如下方式进行import：

```Python
import os
import sys

PARENT_PATH = os.path.dirname(os.path.abspath(__file__))
print PARENT_PATH
sys.path.append('pkgs.zip/libs')

import arrow
```

## 四、To Be Continued...
后面遇到其他问题再更新吧。