LDA主题模型在2002年被David M. Blei、Andrew Y. Ng（是的，就是吴恩达老师）和Michael I. Jordan三位第一次提出，近几年随着社会化媒体的兴起，文本数据成为越来越重要的分析资料；海量的文本数据对社会科学研究者的分析能力提出了新的要求，于是LDA主题模型（Topic Model）作为一种能够从大量文本中提取出主题的概率模型，被越来越多的运用到主题发现、文档标记等社会科学研究中来。

由于国外的例子都是基于英文语言的应用，本着将研究方法本土化的目的，我们将用中文的例子，来分享上述工具的使用。因此，您将从下面的分享中了解到：

- 如何用R对中文文本进行分词？
- 如何用R做主题模型建模？
- 如何用LDAvis对建模的结果进行可视化展现？
- 需要用到的R包有：jiebaR、lda、LDAvis

### jiebaR中文分词
jiebaR是“结巴”中文分词（Python）的R语言版本，支持最大概率法（Maximum Probability），隐式马尔科夫模型（Hidden Markov Model），索引模型（QuerySegment），混合模型（MixSegment）共四种分词模式，同时有词性标注，关键词提取，文本Simhash相似度比较等功能。项目使用了Rcpp和CppJieba进行开发。目前托管在GitHub上。安装很简单。jiebaR中文分词——R的灵活，C的效率

是的,笔者用过jiebaR后，发现确实很方便。jiebaR是使用RCpp开发的，据说jiebaR的分词速度是其他R语言分词包的5-20倍。

```
library(jiebaR) #加载
cutter <- worker(bylines = T,user = "./UsrWords.txt",stop_word = "./stopWords.txt") #创建分词器，其中bylines是否按行来分，user用户词典，stop_word停用词典
comments_seg <- cutter["./Mobilecomments.txt"] #文件分词，直接输入文件地址，分完后自动保存成文件
```

注：笔者演示所用数据为jd上某款手机的评论数据，每条评论一行。

上面三行代码完成了主题模型建模前很多的准备步骤，1、分词，2、根据用户词典分词，3、去掉没用的停用词（网上一搜“中文停用词表”就能搜到），执行了这三行命令后，R会自动保存一个分完词的文件在你的工作目录。结果如：

```
还行 慢慢 **惯 功能 便捷
手机 网上 价格 实惠
收到 货 满意 喜欢 值得 购买
```

但是，由于LDA主题模型不是对文本进行运算，在做主题建模前，还需将文本转化为向量，用数字来代替文本，从而方便运算。

```
comments_segged<- readLines("./Mobilecomments.segment.2015-12-17_13_57_30.txt",encoding="UTF-8") #读取分词结果
comments <- as.list(comments_segged) #将向量转化为列表
doc.list <- strsplit(as.character(comments),split=" ") #将每行文本，按照空格分开，每行变成一个词向量，储存在列表里
```

接下来，就像在python下gensim包里的操作一样，我们需要创建一个词典，并给每个词取一个编号：

```
term.table <- table(unlist(doc.list)) #这里有两步，unlist用于统计每个词的词频；table把结果变成一个交叉表式的factor，原理类似python里的词典，key是词，value是词频
term.table <- sort(term.table, decreasing = TRUE) #按照词频降序排列
```

在前面，我们用stop_words去掉了数字、标点符号、虚词等，由于现代汉语里用单字表示一个词语的词已经很少了，这里为了提高建模效果，我们可以将单字去掉，同时也可以把出现次数少于5次的词去掉。

```
del <- term.table < 5| nchar(names(term.table))<2 #把不符合要求的筛出来
term.table <- term.table[!del] #去掉不符合要求的
vocab <- names(term.table) #创建词库
```

接下来，我们要把文本的格式整理成lda包建模需要的格式。lda包对格式有具体要求：

```
[,1] [,2] [,3]
[1,] 8 261 46
[2,] 1 1 1
```

这是一个文档，第一行是词的ID，第二行是词在文档里出现的频次。为了能生成这样的结构，这里我参考了LDAvis作者的代码：

```
get.terms <- function(x) {
index <- match(x, vocab) # 获取词的ID
index <- index[!is.na(index)] #去掉没有查到的，也就是去掉了的词
rbind(as.integer(index - 1), as.integer(rep(1, length(index)))) #生成上图结构}
documents <- lapply(doc.list, get.terms)
```

至此，我们就准备好了做LDA建模需要用的数据了。

### LDA主题建模
这部分没啥好说的，直接上代码：

```
K <- 5 #主题数
G <- 5000 #迭代次数
alpha <- 0.10 
eta <- 0.02
```

这些为LDA建模需要先设置的几个参数，关于alpha、eta的设置和作用，引用梁斌penny的一段话：

> 其中α，大家可以调大调小了试试看，调大了的结果是每个文档接近同一个topic，即让p(wi|topici)发挥的作用小，这样p(di|topici)发挥的作用就大。其中的β，调大的结果是让p(di|topici)发挥的作用变下，而让p(wi|topici)发挥的作用变大，体现在每个topic更集中在几个词汇上面，或者而每个词汇都尽可能的百分百概率转移到一个topic上。

接下来是主题建模的过程，以文本量大小和迭代次数多少，用时会不同，多则几十分钟，少则一两分钟。

```
library(lda)
set.seed(357) 
fit <- lda.collapsed.gibbs.sampler(documents = documents, K = K, vocab = vocab, num.iterations = G, alpha = alpha, eta = eta, initial = NULL, burnin = 0, compute.log.likelihood = TRUE)
```

### LDAvis可视化
用LDAvis做可视化也很简单，只要把相应的参数准备好：

```
theta <- t(apply(fit$document_sums + alpha, 2, function(x) x/sum(x))) #文档—主题分布矩阵
phi <- t(apply(t(fit$topics) + eta, 2, function(x) x/sum(x))) #主题-词语分布矩阵
term.frequency <- as.integer(term.table) #词频
doc.length <- sapply(documents, function(x) sum(x[2, ])) #每篇文章的长度，即有多少个词
```

调用LDAvis:

```
library(LDAvis)
json <- createJSON(phi = phi, theta = theta, 
doc.length = doc.length, vocab = vocab,
term.frequency = term.frequency)#json为作图需要数据，下面用servis生产html文件，通过out.dir设置保存位置
serVis(json, out.dir = './vis', open.browser = FALSE)
```

至此，一个基于网页的可交互的主题模型可视化分析展示就做出来了，servis会在指定的路径下生成文件夹vis，里面包含：

***d3.v3.js  index.html  lda.css  lda.json   ldavis.js***

接着，下载火狐浏览器，在地址栏输入

```
about:config
```

搜索”security.fileuri.strict_origin_policy”,并设置该项为false，重新打开浏览器即可。

这里还会出现一个问题，由于我们做的是中文的主题建模，而这个工具本身在英文环境下创建的，所以没有考虑中文显示的问题。做完上面的步骤后，打开网页，会出现乱码的问题。为了解决乱码的问题，我们需要将其中的lda.json文件的编码改成UTF8格式，你可以手动改，也可以用R来自动改。

```
writeLines(iconv(readLines("./vis/lda.json"), from = "GBK", to = "UTF8"),
       file("./vis/lda.json", encoding="UTF-8"))
```
