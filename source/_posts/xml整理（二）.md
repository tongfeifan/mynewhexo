---
title: xml整理（二）
date: 2014-11-06 02:22:35
tags: [xml, course]
---

*时隔两日，再次来整理XML的内容，似乎间隔有些长。终究还是不够勤快，压抑不住自己爱玩的心。今天在知乎上看到大师兄又写了一篇[答案](http://www.zhihu.com/question/26092705/answer/32989919)。倒不是对所提的问题有何想法，只是大师兄的某些话说到了心坎里：*<!--more-->

>“你只要不想等，不愿信，你站起来，关灯，收拾干净，拍屁股走人，出来你就是铁人，没有人伤得了你，但这应该不是你想要的。

>这世上容易的，就是看破红尘，难的，恰是命里打滚，轻而易举，说一些不痛不痒，都是没想明白。”

*是吧，感情也罢，梦想也罢，默默地等待，默默地付出，只觉得该为某些事情热血地追求。若自己某日，打着看破的幌子，放弃坚持的初衷。那我，还是想要为尘世的纷扰而不得解吧。等吧，等到想要的结果，等到该来的人。*
>“在散场的最后一秒到来之前，空荡荡的剧场里走进来一个人。

>你就一定要说：

>等你很久啦。”

*扯远了，只是想来接着整理我的XML的。今天打算整理DTD, XML schema,这几项属于Schema的内容。*

---

### 何谓schema

schema实际上一种描述语言，一般用于描述DBs或者XML，在此特指对XML的描述。通过Schema，可以描述XML的

- 标签（tag）和属性（attribute）的名字
- 文件的结构：（元素（element）如何嵌套、哪个元素拥有哪个属性）
- 数据：类型（字符串？数字？）

为何需要schema

- XML文档可以有更多可依赖的格式：结构，标签，数据类型。但同时想保持扩展性与灵活性。
- 可以描述怎样的数据是合法的、期待的、事先设定。

对于使用schema有两种使用方式：

- 描述型（descriptive）:为他人描述XML文档，如此，便可知道如何更好地组织数据。
- 规范型（prescriptive）:防止程序调用了错误的XML文档。

---

### 什么是DTD

DTD是在XML1.0时代的标准。DTD并不是一个典型的XML类型的文档，故而具有以下特点：

- 不能被解析为一个DOM (Document Object Model)
- 无法扩展（即不能在一个DTD文档中import另一个DTD文档）
- 易读易写。

### DTD如何描述XML

描述一个逻辑结构的例子：

- elements

```DTD
<!ELEMENT name(#PCDATA)>
<!ELEMENT person(name, address+, email*)>
<!ELEMENT address(city,(nr,street)?)>
```

- attributes
  
```DTD
<!ATTLIST name type(family|personal|place) "personal">
```

此例子意为：元素name可以使用type属性，type可选值为family,personal,place，其中personal为默认值。
据此规范，正确的xml文档：

```XML
<name>Bijan</name>
<name type="personal">Bijan</name>
<name type="family">Parsia</name>
```

不正确的：

```XML
<name type="DontKnow">Bijan</name>
```

由此，我们可以总结，DTD描述的模板如下：

```XML
<!ELEMENT element-name (element-content)>
<!ATTLIST element-name attribute-name attribute-type attribute-value>
```

其中element-content支持使用元素名的正则表达式。

另外DTD还能描述实体引用（类似宏定义）。例子如下：

```DTD
<!ENTITY writer "Donald Duck">
```

### 如何使用DTD

DTD在XML文档中有两种引入方式，一种为内嵌型（internal），另一种外置型（external）。

内嵌型的使用方式：

```DTD
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE cartoon[
<!ELEMENT cartoon (prolog,panels)>
...
<!ELEMENT speech(#PCDATA)>]>
<cartoon copyright="United Feature Syndicate">
    <prolog>
        ...
    </prolog>
</cartoon>
```

外置型使用方式：

```DTD
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE cartoon SYSTEM "cartoon.dtd">
<cartoon copyright="United Feature Syndicate">
    <prolog>
        ...
    </prolog>
</cartoon>
```

当然也可以两者同时使用：

```DTD
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE cartoon SYSTEM "cartoon.dtd"
[<!ATTLIST cartoon oneMore CDATA #IMPLIED>]>
<cartoon copyright="United Feature Syndicate" oneMore="3">
	<prolog>
		...
	</prolog>
</cartoon>
```

### valid(不知如何翻译)

如果XML文件完全满足某一DTD文档，则认为该XML文档 valid w.r.t a DTD。

如果XML同时满足以下条件，则认为XML文档valid：

- 该文档是well-formed
- 该文档引入了一个DTD文档
- 该文档valid w.r.t **引入的**文档
- 声明元素为该文档的根元素：`<!DOCTYPE cartoon SYSTEM "cartoon.dtd">`其中的cartoon。

---

### 什么是XML Schema

XML Schema（也被称为XML Schema Definition、XSD）是DTD的替代品，并且一致认为其比DTD更加成功。相对DTD，XSD有这些特点：

- DTD**不是**well-formed XML文档，而XSD**是**
- XSD有更强的表达能力
- XSD支持namespaces, datatypes(date, 06/11/2014)
- XML能描述元素的复杂content。

### XSD例子：

DTD

```DTD
<?xml version="1.0" encoding="UTF-8"?>
<!ELEMENT note (to, from, sentOn, heading, body)>
<!ELEMENT to (#PCDATA)>
<!ELEMENT from (#PCDATA)>
<!ELEMENT sentOn (#PCDATA)>
<!ELEMENT body (#PCDATA)>
```

相应XSD

```XML
<?xml version="1.0"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" targetNmaespace = "http://www.w3schools.com" xmlns="http://www.w3schools.com" elementFormDefault="qualified">
	<xs:element name = "note">
		<xs:complexType>
			<xs:sequence>
				<xs:element name="to" type="xs:string" />
				<xs:element name="from" type="xs:string" />
				<xs:element name="sentOn" type="xs:date" />
				<xs:element name="body" type="xs:string" />
			</xs:sequence>
		</xs:complexType>
	</xs:element>
</xs:schema>
```
###type, content, restriction and extension
XSD中的type可分为simpleType and complexType:

- simpleType可以用于属性，不带子元素或者不带属性的元素
- complexType可以用于element content 或者mixed element content 或者 text content和attribuctes

而content的使用：

- 对于有属性声明的元素，我们不能使用simpleType，但是我们可以使用simpleContent来通过继承方式获得simpleType:

```xml
<xs:element name = "size">
	<xs:complexType>
		<xs:simpleCont>
			<xs:extension base = "xs:integer">
				<xs:attribute name = "country" type = "xs:string" />
			</xs:extension>
		</xs:simpleCont>
	</xs:complexType>
</xs:element>
```

- complexContent则包含了继承或扩展自complexType的content或element。

而restriction and extension则用于继承或者扩展type。从字面意义便可理解两者的区别。

更详细的内容可以参看[w3schools.com](http://www.w3schools.com/schema/schema_elements_ref.asp)。