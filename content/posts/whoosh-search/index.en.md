---
title: Whoosh+jieba 中文检索
date: 2017-04-26 03:33:27
comments: true
weight: 5
draft: false
categories: ["Coding"]
tags: ["Python", "Whoosh"]
lightgallery: true
---

## Whoosh + jieba 中文检索

### 背景

最近项目要用到 [Whoosh](https://whoosh.readthedocs.io/) 一个 Python 编写的索引检索模块，发现比较少中文资料并且看了学长的代码也好多不懂，故自己照着官网文档撸了一遍，把我自己的理解和官网一些不太清楚的解释写下来。

<!--more-->

### 快速上手

#### 几个核心对象

##### Index 和 Schema 对象

 在使用 Whoosh 前，首先需要创建的就是 *index* 对象，*index* 对象是一个全局索引。在创建 *index* 对象前首先要声明 index 对象的一些属性，所以需要在创建一个用于包装这些属性的 *schema* 对象。*schema* 有很多 Fields(一个 Field 是 index 对象的一个信息块，即需要被我们检索的内容)

举个栗子，以下代码创建了一个包含 "title" 和 "path" 和 "content" 三个 Fields 的 *schema* 对象

```python
from whoosh.fields import Schema, TEXT, ID
schema = Schema(title=TEXT, path=ID, content=TEXT)
```

创建 *schema* 对象时需要用关键字来映射 Field name 和 Field type，如上的 title=TEXT

一旦创建好了 *schema* 对象，接着就是使用 create_in 方法来创建 *schema* 的索引

```python
import os.path
from whoosh.index import create_in

if not os.path.exists("index"):
	os.mkdir("index")
idx = create_in("index", schema)
```

接着可以用以下两种方法打开一个已创建的索引

```python
# 方法一 使用FileStorage对象
from whoosh.filedb.filestore import FileStorage
storage = FileStorage(idx_path)  #idx_path 为索引路径
idx = storage.open_index(indexname=indexname, schema=schema)

# 方法二 使用open_dir函数
from whoosh.index import open_dir
idx = open_dir(indexname=indexname)  #indexname 为索引名
```

#### IndexWriter 对象

一旦有了 *index* 对象，我们就需要在 index 里写入需要被检索的信息，所以 IndexWriter 对象就是用来提供一个 add_document(**kwargs) 方法来在之前声明的各种 Fields 里写入数据

```python
writer = idx.writer()  #IndexWriter对象
writer.add_document(
	title=u"Document Title",
    path=u"/a",
    content=u"Hello Whoosh"
)  # Field 和 schema 中声明的一致
writer.commit()  # 保存以上document
```

需要注意的是：

- 不是每个 Field 都要赋值
- Field 传值一定是 unicode 类型的值

如果有一个 Field 同时要被当做索引并保存之，那么可以用一个 unicode 值来做索引同时保存另一个对象

```python
writer.add_document(title=u"Title to be indexed", _stored_title=u"Stored title")
```

如果需要异步处理可以创建异步的 IndexWriter 对象

```python
from whoosh.writing import AsyncWriter
writer = AsyncWriter(index=index)
```

如果需要Buffer进行处理可以创建 BufferedWriter 对象

```python
from whoosh.writing import BufferedWriter
# period是多次commit的最大间隔时间，limit是需要缓存的最大数量
writer = BufferedWriter(index=index, period=120, limit=20)
```

##### Searcher 对象

在开始搜索索引之前，我们需要创建 *searcher* 对象

```python
searcher = idx.sercher()
```

但是一般来说不会这么创建搜索器 *searcher* ，这样做没法来索引检索完成后关闭搜索器释放内存(只要知道 searcher 很吃内存就行)，我们一般用 with 来创建 *searcher* 对象从来保证搜索器使用完毕后可以被正确关闭

```python
with idx.sercher() as searcher:
    ...
```

以上写法等同于

```python
try:
	searcher = idx.searcher()
	...
finally:
	searcher.close()
```

搜索器的 ```search()``` 方法需要传入一个 *Query* 对象，我们可以直接构造一个 *Query* 对象或者使用 *query parser* 来解析一个查询字段

举个栗子

```python
# 直接构造查询对象
from whoosh.query import *
myquery = And([Term("content", u"apple"), Term("content", "bear")])
```

默认的 *QueryParser* 允许使用查询原语 AND 和 OR 和 NOT 就像 SQL 一样简单！

```python
# 使用解析器解析查询字段
from whoosh.qparser import QueryParser
parser = QueryParser("content", idx.schema)
myquery = parser.parse(querystring)
```

构造完查询对象后，就可以使用搜索器的 search() 方法来进行检索

```python
results = searcher.search(myquery)
print(results[0])
{"title": "Document", "content": "Hello Whoosh"}
```

更通常的我们使用分页查询 search_page() 的方法

```python
results = searcher.search_page(myquery, page_num, page_len)
```

#### 结合 jieba 分词使用

Whoosh 的基本用法如上，接着我要在 QueryString 中加入结巴分词分析模块

由于 jieba 0.30 之后的版本已经添加用于 Whoosh 的分词接口: ChineseAnalyzer, 所以还是很方便的

首先在 Whoosh schema 对象的创建的 whoosh.fields.TEXT，默认的声明 TEXT 时字段的 FieldAttributes 默认有个属性 analyzer

analyzer 是一个带有 *__call__* 魔术方法的类，用来进行 TEXT 词域的分析，在调用时会把 TEXT 域里的值进行 *__call__* 处理

analyzer 接收的参数是一个 unicode 字符串，返回值是字符串切分，举个栗子

e.g.(

​	param = "Mary had a little lamb"

​	return = ["Mary", "had", "a", "little", "lamb"]

)

使用的是 Whoosh 的 StandardAnalyzer ，是英文的分词器。为了对接上 jieba，做中文分词，需要把 TEXT(analyzer=analysis.StandardAnalyzer()) 换成 jieba 的 ChineseAnalyzer 即可

```python
from __future__ import unicode_literals
from jieba.analyse import ChineseAnalyzer

analyzer = ChineseAnalyzer()

schema = Schema(title=TEXT(stored=True), path=ID(stored=True), content=TEXT(stored=True, analyzer=analyzer))

idx = create_in("test", schema)
writer = idx.writer()
writer.add_document(
	title="test-document",
    path="/c",
    content="This is the document for test"
)
writer.commit()
searcher = idx.searcher()
parser = QueryParser("content", schema=idx.schema)

for keyword in ("水果","你","first","中文","交换机","交换"):
    print("result of ",keyword)
    q = parser.parse(keyword)
    results = searcher.search(q)
    for hit in results:
        print(hit.highlights("content"))
    print("="*10)
```

还是很方便的。

