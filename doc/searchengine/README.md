# 搜索引擎

搜索引擎大致可以分为四个部分：搜集、分析、索引、查询


### 搜集


#### 爬取链接文件：links.bin

首先需要利用爬虫爬取网页，搜索引擎使用广度优先的方式，把爬取到的地址存放到队列当中。由于地址越来越多，
内存会存不下，需要把地址写入到硬盘当中，生成链接文件 links.bin ,从 links.bin 文件中读取链接地址进行爬取，
解析页面中的地址存放到links.bin文件当中。

#### 链接地址判重：bloom_filter.bin

使用[布隆过滤器](https://github.com/lzle/algorithm/tree/master/datastructure/bitmap)的方式，针对链接地址去重，并把数据持久化到 bloom_filter.bin 文件当中。

当服务器宕机时，可以将 bloom_filter.bin 文件恢复到内存当中，不至于造成数据丢失。

#### 网页内容存储：doc_raw.bin

爬取到的网站内容，需要进行存储，用以后续的数据分析、索引建立。将多个网页内容存储到一个文件中，每个文件的大小不能超过一定的值（比如 1GB)。

每个网页之间，通过一定的标识进行分隔，方便后续读取，如下

<img src="https://static001.geekbang.org/resource/image/19/4d/195c9a1dceaaa9f4d2483fa91455404d.jpg" width=500>


#### 网页链接及其编号的对应文件：doc_id.bin

对网页链接进行编号，方便我们后续对网页进行分析、索引。

维护一个中心的计数器，每爬取到一个网页之后，就从计数器中拿一个号码，分配给这个网页，然后计数器加一。
在存储网页的同时，我们将网页链接跟编号之间的对应关系，存储在另一个 doc_id.bin 文件中。

### 分析

离线网页分析分为两部分，抽取网页文本信息，建立临时索引。

#### 抽取网页文本信息

网页内容中去掉JavaScript 代码、CSS 格式、 HTML 标签。使用AC自动机多模式串匹配。

#### 临时索引

针对文本信息建立临时索引，中文分词有一种比较简单的思路，基于字典和规则的分词方法。

> 其中，字典也叫词库，里面包含大量常用的词语（我们可以直接从网上下载别人整理好的）。我们借助词库并采用最长匹配规则，来对文本进行分词。所谓最长匹配，
>也就是匹配尽可能长的词语。我举个例子解释一下。比如要分词的文本是“中国人民解放了”，
>我们词库中有“中国”“中国人”“中国人民”“中国人民解放军”这几个词，那我们就取最长匹配，
>也就是“中国人民”划为一个词，而不是把“中国”、“中国人”划为一个词。具体到实现层面，
>我们可以将词库中的单词，构建成 Trie 树结构，然后拿网页文本在 Trie 树中匹配。


分词完成之后，我们都得到一组单词列表。把单词与网页之间的对应关系，写入到一个临时索引文件中（tmp_Index.bin）

<img src="https://static001.geekbang.org/resource/image/15/1e/156ee98c0ad5763a082c1f3002d6051e.jpg" width=500>

term_id 为单词的编号，可以节省存储空间，单词编号，可以参考网页编号，使用计数器分配。已存在的单词编号，使用
散列表存储，相同的单词不用再进行编号分配。

上面工作完成后单词和编号的对应关系写入到 term_id.bin 中。

### 索引

索引阶段主要负责将分析阶段产生的临时索引，构建成倒排索引。

在临时索引文件中，记录的是单词编码与单个网页编码的对应关系，需要转为为单词编码与多个网页编码对应。

<img src="https://static001.geekbang.org/resource/image/15/1e/156ee98c0ad5763a082c1f3002d6051e.jpg" width=500>

考虑到临时索引文件很大，无法一次性加载到内存中，搜索引擎一般会选择使用多路归并排序的方法来实现。

按编码顺序建立倒排索引。

<img src="https://static001.geekbang.org/resource/image/c9/e6/c91c960472d88233f60d5d4ce6538ee6.jpg" width=500>

除了倒排文件之外，我们还需要一个文件，来记录每个单词编号在倒排索引文件中的偏移位置。
我们把这个文件命名为 term_offset.bin。这个文件的作用是，帮助我们快速地查找某个单词编号在
倒排索引中存储的位置，进而快速地从倒排索引中读取单词编号对应的网页编号列表。

<img src="https://static001.geekbang.org/resource/image/de/54/deb2fd01ea6f7e1df9da1ad3a8da5854.jpg" width=500>


### 查询

前面三个阶段的处理，只是为了最后的查询做铺垫。因此，现在我们就要利用之前产生的几个文件，来实现最终的用户搜索功能。

* doc_id.bin：记录网页链接和编号之间的对应关系。

* term_id.bin：记录单词和编号之间的对应关系。

* index.bin：倒排索引文件，记录每个单词编号以及对应包含它的网页编号列表。

* term_offsert.bin：记录每个单词编号在倒排索引文件中的偏移位置。

这四个文件中，除了倒排索引文件（index.bin）比较大之外，其他的都比较小。为了方便快速查找数据，我们将其他三个文件都加载到内存中，并且组织成散列表这种数据结构。

当用户在搜索框中，输入某个查询文本的时候，我们先对用户输入的文本进行分词处理。假设分词之后，我们得到 k 个单词。

我们拿这 k 个单词，去 term_id.bin 对应的散列表中，查找对应的单词编号。
经过这个查询之后，我们得到了这 k 个单词对应的单词编号。

我们拿这 k 个单词编号，去 term_offset.bin 对应的散列表中，查找每个单词编号在倒排索引文件中的偏移位置。经过这个查询之后，我们得到了 k 个偏移位置。我们拿这 k 个偏移位置，去倒排索引（index.bin）中，查找 k 个单词对应的包含它的网页编号列表。

经过这一步查询之后，我们得到了 k 个网页编号列表。我们针对这 k 个网页编号列表，统计每个网页编号出现的次数。

具体到实现层面，我们可以借助散列表来进行统计。统计得到的结果，我们按照出现次数的多少，从小到大排序。出现次数越多，说明包含越多的用户查询单词（用户输入的搜索文本，经过分词之后的单词）。

经过这一系列查询，我们就得到了一组排好序的网页编号。我们拿着网页编号，去 doc_id.bin 文件中查找对应的网页链接，分页显示给用户就可以了。






