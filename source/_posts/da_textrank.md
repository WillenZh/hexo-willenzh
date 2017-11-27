title: 基于TextRank的中文摘要工具
tags:
  - Python
  - NLP
categories:
 - 机器学习
date: 2016-05-11 17:08:00
---
最近需要做一些文本摘要的东西，选取了TextRank（论文参见[《TextRank: Bringing Order into Texts》](http://web.eecs.umich.edu/%7Emihalcea/papers/mihalcea.emnlp04.pdf)）作为对比方案，该方案可以很方便的使用Python相关库进行实现。

下面介绍如何利用Python实现一个简单的文本摘要工具。
<!--more-->
### [Demo](http://www.jayveehe.com/da_demo)

---

## 【前期准备】：
1. Python 2.7.x - 当然也推荐Python3，少掉很多编码问题。信仰选2！
2. [jieba分词](https://github.com/fxsjy/jieba) - 最好的python中文分词工具（最新清华出了个[THULAC](http://thulac.thunlp.org)，有兴趣的可以试试，看对比效果似乎更好）
3. [networkx](https://github.com/networkx/networkx) - 一个非常棒的复杂网络工具库

## 【背景知识】

利用Textrank做文本摘要的核心思想很简单，和著名的网页排名算法[PageRank](https://zh.wikipedia.org/wiki/PageRank)类似：每个句子可以作为一个网络中的节点（称为`节点i`），与之相连的其他节点（例如`节点j`）会对其`重要度`产生一定的“贡献值”，该“贡献值”与`节点j`自身的`重要度`以及i、j之间的`相似度`（也可以称为连接的强度）有关，只需要对整个图进行迭代直至收敛，最后各节点的分值即是该句子的重要性，根据重要性排序后选取前k个句子即可作为摘要。

## 【基于Textrank的文本摘要实现步骤】
### 一、对文本进行分句
假设只处理中文文本，则采取句号或换行符作为分句准则：
```python
def split_sentences(full_text):
    sents = re.split(u'[\n。]', full_text)
    sents = [sent for sent in sents if len(sent) > 0]  # 去除只包含\n或空白符的句子
    return sents
```

### 二、句子相似度的计算
Textrank的原始论文中，句子相似度是基于两个句子的共现词的个数计算的，在此沿用论文的公式：

$$Similarity(S_i,S_j)=\\frac\{\\{w_k|{w_k}\\in\{S_i\}\\&\{w_k\}\\in\{S_j\}\\}\}\{log(S_i)+log(S_j)\}$$

实现时，采用共现词计数进行相似度计算，输入的是每个句子的terms，以避免重复分词。代码如下：
```python
def cal_sim(wordlist1, wordlist2):
    """
    给定两个句子的词列表，计算句子相似度。计算公式参考Textrank论文
    :param wordlist1:
    :param wordlist2:
    :return:
    """
    co_occur_sum = 0
    wordset1 = list(set(wordlist1))
    wordset2 = list(set(wordlist2))
    for word in wordset1:
        if word in wordset2:
            co_occur_sum += 1.0
    if co_occur_sum < 1e-12:  # 防止出现0的情况
        return 0.0
    denominator = math.log(len(wordset1)) + math.log(len(wordset2))
    if abs(denominator) < 1e-12:
        return 0.0
    return co_occur_sum / denominator
```

### 三、运用networkx工具库进行pagerank迭代
利用`networkx`库创建一个graph实例，调用networkx的pagerank方法对graph实例进行处理即可，需要注意的是，networkx有三种pagerank的实现，分别是`pagerank`、`pagerank_numpy`和`pagerank_scipy`，从名称可以看出来它们分别采用了不同的底层实现，此处我们任选其一即可。代码如下：

```python
def text_rank(sentences, num=10, pagerank_config={'alpha': 0.85, }):
    """
    对输入的句子进行重要度排序
    :param sentences: 句子的list
    :param num: 希望输出的句子数
    :param pagerank_config: pagerank相关设置，默认设置阻尼系数为0.85
    :return:
    """
    sorted_sentences = []
    sentences_num = len(sentences)
    wordlist = []  # 存储wordlist避免重复分词，其中wordlist的顺序与sentences对应
    for sent in sentences:
        tmp = []
        cur_res = jieba.cut(sent)
        for i in cur_res:
            tmp.append(i)
        wordlist.append(tmp)
    graph = np.zeros((sentences_num, sentences_num))
    for x in xrange(sentences_num):
        for y in xrange(x, sentences_num):
            similarity = cal_sim(wordlist[x], wordlist[y])
            graph[x, y] = similarity
            graph[y, x] = similarity

    nx_graph = nx.from_numpy_matrix(graph)
    scores = nx.pagerank(nx_graph, **pagerank_config)  # this is a dict
    sorted_scores = sorted(scores.items(), key=lambda item: item[1], reverse=True)

    for index, score in sorted_scores:
        item = {"sent": sentences[index], 'score': score, 'index': index}
        sorted_sentences.append(item)

    return sorted_sentences[:num]
```

上面的`text_rank `函数返回的结果中即包含了num句关键句子，可以作为组成摘要的基础。

### 四、重新组织摘要
由`text_rank`得到的前k个句子，只能表示这几个句子很重要，然而他们在逻辑上很难串联起来。如何重组织摘要，在学术界也是一大研究热点。根据不同的处理粒度（句子级、字词级）和不同的处理思路（根据语义重组还是改变现有词句的顺序），生成的摘要在阅读性上有很大的不同。
在此为了简便，选取最简单的，根据句子在文章中出现的顺序对`text_rank `结果进行重排序。代码如下：
    
```python
def extract_abstracts(full_text, sent_num=10):
    """
    摘要提取的入口函数，并根据textrank结果进行摘要组织
    :param full_text:
    :param sent_num:
    :return:
    """
    sents = split_sentences(full_text)
    trank_res = text_rank(sents, num=sent_num)
    sorted_res = sorted(trank_res, key=lambda x: x['index'], reverse=False)
    return sorted_res
```

---
        
#### **至此，一个精简版的基于TextRank的文本摘要工具就完成了。**

## 【测试结果】
选取好奇心日报上的一则新闻《[中国公司在和 AC 米兰谈收购，但成不成是另一回事](http://www.qdaily.com/articles/26710.html)》，将文本输入`extract_abstracts()`：
```python
if __name__ == '__main__':
    raw_text = codecs.open('/Users/jayvee/github_project/DocumentsAbstractProject/static/text', 'r', 'utf8').read()
    res = extract_abstracts(raw_text, sent_num=5)
    for s in res:
        print s['score'], s['sent']
```
所得结果为：  

    0.0742110929605 传了两个月的中国公司要收购 AC 米兰的事情终于有了一个确切的消息，拥有 AC 米兰俱乐部股权的 Fininvest 公司官方正式确认正在和一家来自中国的企业商谈俱乐部股权出售事宜
    0.0697076855554 2015 年 11 月，AC 米兰老板贝卢斯科尼访华，并称就美丽之冠绿卡收购 AC 米兰一定数量的股权一事达成了合作意向，然而这件事也就此没了下文
    0.0693598701975 一年多前，曾有泰国财团为 AC 米兰开出了 5 亿欧元收购 48% 的股份的价码，这意味着当时 AC 米兰的估值为 10 亿欧元
    0.0685480949813 根据 AC 米兰官网上的数据，这家俱乐部从 2007 年开始就一直处在净亏损的状态中，2014 年的亏损额接近 1 亿欧元，更是创下了历史新高
    0.0701256108461 如果 AC 米兰真的被中国人买下来了，那么在这个买家看来，他买下来的也绝不仅仅只是一个足球俱乐部而已
    

~~看上去还行~~

## 【TODOs&相关阅读】
1. 可以看出，本套方案第一个可修改的地方是句子间相似度的计算，如何精确衡量句子之间的语义相似度，是NLP的另一个坑。（可以用TF-IDF或词向量相加的方式生成句子向量并求cosine相似度，或者一些基于神经网络的方法）
2. TextRank同期的也是基于图结构的rank算法[LexRank（参见论文）](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiA8MiKttLMAhVS0WMKHfLcDSIQFggdMAA&url=http%3A%2F%2Fwww.cs.cmu.edu%2Fafs%2Fcs%2Fproject%2Fjair%2Fpub%2Fvolume22%2Ferkan04a-html%2Ferkan04a.html&usg=AFQjCNHDCgWWXl83W7qvBe5FCTRvNGyR6g&sig2=7Fbk5oymRFalJO2spjJ4dA)即是在相似性的计算上有所区别，采用了一些词法上特征进行相似度计算，有兴趣的读者可以参考。


---

*转载请注明原处
http://jayveehe.github.io/2016/05/11/da_api_demo/*