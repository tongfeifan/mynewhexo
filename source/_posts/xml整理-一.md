---
title: xml整理(一)
date: 2014-11-03 18:02:45
tags: [xml, course]
---


*Github博客在第一次建立之后，一直在赶作业，直到今天早上提交了两个deadline，终于松了一口气。这里接下来将是个人记录学习，以及一些想法的地方。目前计算机的课Semi-structured Data and the Web 和商学院的课Entrepreneurial Commercialisation of Knowledge 都已经结课，接下来的时间里将整理一下两门课的内容，若有空闲，则同时整理一下其他课。*<!--more-->

---

### 何谓Semi-structured data
根据[维基百科](http://en.wikipedia.org/wiki/Semi-structured_data)，semi-structured data定义如下：

> Semi-structured data is a form of structured data that does not conform with the formal structure of data models associated with relational databases or other forms of data tables, but nonetheless contains tags or other markers to separate semantic elements and enforce hierarchies of records and fields within the data. Therefore, it is also known as sef-desribing structure.

简单说便是，半结构化数据不同于关系型数据库中的正规数据结构，而是通过tag或其他标记来区分数据元素与等级。
在[维基百科](http://en.wikipedia.org/wiki/Semi-structured_data#Types_of_Semi-structured_data)中，Semi-structured data的类型分别有XML和JSON，XML作为最经典，曾经最流行的半结构化语言，被广泛应用于各种网络服务中，而JSON则因为其可读性更强，逐渐作为XML的替代品。这里主要讨论的是XML相关技术。

---

### xml相关技术内容

Semi-structured Data and the Web课程讲述的内容是以xml为核心的所有围绕xml展开的技术。其中涉及到的技术内容包括：
![XML map](http://img02.taobaocdn.com/imgextra/i2/439962795/TB2msLEaVXXXXa_XXXXXXXXXXXX_!!439962795.png)
在接下来的内容中，会分步逐渐解释每一项技术的内容。首先提到的自然是核心技术XML

---

### 什么是XML
- XML是一个W3C标准
- XML被设计成**simple**, **generic** and **extensible**
- XML文件可描述**structure**和**data**，并可关联于一个**DOM tree**
- XML由element分割成更小的部分，element可包含element，elment之间存在无歧义的等级制度。
- 一个XML文件有一个root element和其他element组成

以下为一个典型的XML文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE cartoon SYSTEM "cartoon.dtd">
<cartoon copyright="United Feature Syndicate" year="2000">
    <prolog>
        <series>Dilbert</series>
        <author>Scott Adams</author>
        <characters>
            <character>The Pointy-Haired Boss</character>
            <character>Dilbert</character>
        </characters>
    </prolog>
    <panels>
        ...
    </panels>
</cartoon>
~~~

其中第1行代码`<?xml version="1.0" encoding="UTF-8"?>`为XML声明，该声明定义了XML版本为1.0，字符编码UTF-8。第2行代码`<!DOCTYPE cartoon SYSTEM "cartoon.dtd">`为XML 类型声明（可选），其中引用一个称为Document Type Definition的语法描述文件；`cartoon`为必须为文件的root element。

### XML elements
element 定义

- element由标签界定
- 标签在尖括号中
- 标签区分大小写。e.g., <FROM>和<from>是不同的
- 标签可分为开始标签<...>和结束标签</from>
- element有一对开始标签与结束标签所界定
- element具有属性(attribute)。e.g., 

```xml
<cartoon copyright="United Feature Sndicate">
```

element概念


```xml
<element-name attr-decl1 ... attr-decln>
    content
</element-name>
```

- 允许多个属性。
- 属性声明格式`attr-name="attr-value"`。
- 一个element中，每个属性最多只允许出现一次。
- content可以为**空**<1>，可以为**text**<2>，可以为一或多个**element**<3>。
其中<2>称为simple content，<3>称为element content，<2>+<3>称为mixed content。
- 空元素可以写做。

```xml
<element-name attr-decl1 ... attr-decln />
```

Well-formed XML document
- 只有一个root element。
- 标签,<,>是正确的（包括在字符串数据中没有<或&）。
- 标签以正确的方式嵌套。
- 对每一个标签来说，属性都是唯一的，且属性值在引号内。
- 标签内没有注释。
Well-formed是一个XML最基本的要求，只有如此，XML才能被解析成树。

---

*今天先整理XML文件的相关内容，下一次我们来讲一讲关于 Schema的相关内容，其中将会涉及到DTD、XML Schema、RelaxNG。*