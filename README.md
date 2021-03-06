
## 一、项目概述

**1.业务描述**  
项目主要前台使用Flask+Bootstrap+d3+mysql，后端使用的算法模型搭建工具包括了结巴分词、哈工大pyltp语言模型,对输入新闻进行人物言论提取。同时使用了scrapy+mysql+redis实现增量去重采集人民网、新华网新闻数据，提供新闻中的关键词数据完成词云功能展示、知识图谱展示。

## 二、项目主要功能描述

**1.新闻言论提取**  
+ 输入：通过前端页面输入新闻内容，并通过ajax提交中api.py文件中的parse_sentence()函数进行处理  
+ 处理：paese_sentence()主要调用controller/Model.py文件中的模块进行处理，并调用mysql数据库进行数据操作（select、insert）  
+ 输出：由parse_sententce()返回数据，由前端ajax接收数据  
**2.词云展示**  
+ 输入：无  
+ 处理：time.py主要为定时任务，每天24点自动执行一次update_wordcloud_img()。  
+ 输出：前端页面index.html中img标签直接调用‘/static/images/word_cloud.png’路径展示图片。  
**3.知识图谱展示**   
+ 每条新闻采集5条关键词，查询所有关键词展示成知识图谱的形式  
+ 输入：无  
+ 处理：有前台ajax请求，api.py中的knowlege_graph()负责查询mysql中的数据进行处理
+ 输出：knowlege_graph()返回数据，ajax接收数据，d3模块（index.html中draw函数）进行数据展示  
**4.新闻采集（后台）**  
+ 输入：根据规律整理好人民网、新华网的列表类网址，通过网址向下采集详情页，文件位于/news_spider/start_urls.py中  
+ 处理：调用scrapy框架采集新闻数据里，同时此新闻采集为定时功能，每次进行新的采集，从mysql中查询采集过的url，在redis中进行缓存，防止重复采集  
+ 输出：采集完成的新闻数据，存储在mysql中的news表中  
**5.定时任务（timer.py）**  
+ word_sim()：对采集后的新闻进行处理并存储到mysql中的word_sim表中。主要存储内容包括：人物言论(name_says)、新闻中关键字(name_entity)、新闻链接（url）、所属栏目（category）、新闻内容（content）、says_entity(废弃未使用)  
+ update_wordcloud_img()：如前述，主要更新词云图片  
**6.算法模型（Model.py）**  
+ sentence_process(sentence):输入文章，对文章分段、分句，由valid_sentences(sentence_list)进一步处理，返回人物及言论
+ valid_sentences(sentence_list)：对单个段落进行处理并返回人物言论，内部使用sentence_embedding进行相邻句子关联，主要涉及了项目指导中的Stanford相邻句子关联度的论文，使用皮尔斯系数（pearsonSimilar(inA,inB)）进行相似度判断。单个句子进一步由single_sentence（sentence,just_name,ws）处理。
+ single_sentence（sentence,just_name,ws）：提取单个句子人物及言论。**但是言论内部的言论未提取，例如：新华社报道，习大大表示XXXXXX。**  
**7.其他**
+ 其中说近似词提取以及语言模型训练工作在/app/controller/Pretreatment.py中生成，**说的近似词提取为老师课堂模板，未进行优化**。

## 三、项目部署

项目部署至Ubuntu参考地址：https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04

项目启动文件wsgi.py中from run import app,在测试过程中无法正常使用，按照网上意见改成from run imort app as application,同时下方的app.run()改成application.run

ltp模型为哈工大的语言模型，model自己训练模型。训练方法在/app/controller/Pretreatment.py中，分别使用ltp与model文件夹分别放在与run.py同级目录

## 四、经验教训

+ **哈工大语言模型安装：**尝试用python3.7版本安装未成功，后来发现平台最高支持3.6版本。  
安装过程：https://blog.csdn.net/laoyaotask/article/details/45312905 （原文出处找不到了）
+ **哈工大与jieba分词进行比较：**在Model.py文件中使用jieba_compare_pyltp()临时函数对两个工具交叉功能进行比较，发现pyltp分词效果好于jieba，但缺点是执行效率慢。
+ **算法模型执行速度慢：**在打印每个函数执行时间的过程中发现get_count()函数执行次数非常多，它主要为实现Stanford的句子相似度算法，计算句子中每个词语的词频及词向量，每篇文章都要重复调用几十次以上。同时，发现哈工大的分词工具pyltp.segment执行每次在3秒以上，相当耗时。
+ **普林斯顿语句近似度算法**：按照论文算法只完成了Vs=1/|s|*∑a/(a+p(w))*Vw，**后半部分的矩阵运算不知道该用如何工具去运算**。
+ **其他：**在项目开发过程中，加了一些功能点在上面，导致了项目最后头重脚轻，前端后台的工作做了很多，但是算法模型部分没有得到很好的完善。甚至在开发的过程中已经记不清当初要做一个什么东西，做这个东西的目的是什么，只是觉得该加就加上去了。这在项目管理中是不允许的，应事先确定好需求和计划，然后再去执行，等到基本需要功能完成后再去进行扩展。

## 五、更新记录

**杨奕康：5月27日更新:**
* 更新分句规则：
    0. **假设**: 
        * 同一言论不会横跨两个段落，我们认为可以以段落为单位分批处理。
        * 代词所指的发言人出现在上文，我们暂时假设为是上一个发言人。
    
    1. 以引号（“”），句号（。），感叹号（!），问号（？）为基准对句子进行分割获得片段，
        * 在引号包含的范围内不进行分割，则不对其进行分割，
        * 仅当右引号（”）左边紧邻句号时，即（。”）才对右引号进行分割。
        
    2. 获得片段后，对每一个片段进行**依存分析**并基于主谓搭配获得name(名字)和 saying(言论)。当遇到以下情况时
        * 若句子片段是左引号（“）开头，则假定该片段为主语后置的**直接引用**，依照正则表达式提取分引号包裹部分提取name，其余引号包裹部分默认为言论。
        * 若句子片段中存在‘：’，则‘：’右边部分全看作saying，并假设左边最近的一个名词（ns, ni）为name。
        * 若无法提取name以及saying，基于句子向量判断是否与前一发言人言论相似，若相似则与之前言论合并，反之则跳过。
        * 若提取出的name为代词，同样基于句子向量判断是否与前一发言人言论相似，相似则合并，反之跳过。
        
* 后续调整：
    1. 代词部分假设不够严谨，下一步会着重于前一个言论中提到的人。
    2. 句子向量部分不够精简，计算耗时太多。
    3. 代码中函数部分重合功能太多，重复计算较多。
