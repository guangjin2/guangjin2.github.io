---
title: step文件初探
date: 2022-11-05 00:00:00
tags: 
  - step
categories: 工业建模
top_img: /img/mou.jpeg
cover: false
copyright: true
toc_number: false
---

> **声明**：本文简单介绍step 文件的基础知识

### 1. 什么是STEP标准
STEP 全称Starndard for Exchange of Product Model Data,即产品模型数据交换规范，代号为**ISO-10303**. 它提供一种不依赖具体系统的中性机制，旨在实现产品间数据的交换和共享，从而提高产品的开发效率。
比如说芯片设计环节可能先建模，再对模型求解，此时就可能需要在建模软件和求解软件这两种产品间传递模型数据。目前工业应用中涉及模型的部分基本会兼容这种格式，比如CAD, HFSS等。 

### 2. STEP标准层次结构
STEP 系统主要分为三个层次， 如图（出自参考资料④）

![step](assets/StepIntro/step.png)

>每part对应一个ISO文档，每个ISO 文档都要收费，不过 [道巴客客](https://www.doc88.com) 还能看到免费的文档， 国标也有一些协议会直接引用这些ISO协议（国标对于我们是免费查看的）


一般而言，我们只需要关注应用层协议即可， 主流的建模协议主要如下（出自参考资料 ⑤ ，其各自特性可以前往参考， 目前最广泛的应该是AP214）：

+ AP203:Configuration controlled 3d design of mechanical parts and assemblies.
+ AP214:Core data for automotive mechanical design processes.
+ AP242：Managed model-based 3D engineering.

现在我们探讨step文件的话，也就是说实现方法为P21，假设应用层协议使用AP214，描述方法为P11，那么三者的关系可以简单的理解为：
P11 描述Express语法
P21 描述.step 的文件格式及其与P11的转换
P214 使用P11定义了一系列汽车行业的实体

如果P214中定义了车如下
```auto
Entity car
 year: Integer
 color: COLOR
Entity

Entity color
  name: string
Entity

```

当我们设计两辆车之后, 保存为step文件
```auto
HEADER
...
ENDSEC
DATA
#1=CAR(1993，#2)
#2=COLOR('red')
#3=Car(1994，#2)
ENDSEC
```

> STEP文件分为头部和数据段，应用协议其实就是定义数据段的实体（'#'代表一个实体），也就是schema

### 3. STEP文件格式
上文介绍了step文件的生成， 以下简单介绍一下格式，用<b>//</b>和<b>/**/</b>注释说明，实际文件不会出现这种标志
```c++
ISO-10303-21; //表明文件协议类型，文件格式实际定义在ISO-10303-21 Table3
HEADER;	//头段开始, 每个文件有且仅有一个头段，但可以有多个数据段
/*
 在ISO-10303-21搜索file_description对应8.2.1节定义如下
 
 ENTITY file_description;
  description : LIST [1:?] OF STRING (256) ; //至少有一个值的string list
  implementation_level : STRING (256) ;//这个等级需要对应文档确定
 END_ENTITY;
 
 其余头部实体均可类似查找, Express语法不明确时可以参照ISO-10303-11
*/
FILE_DESCRIPTION(('FreeCAD Model'),'2;1'); //（）表示list
FILE_NAME('Open CASCADE Shape Model','2022-11-19T22:00:49',('Author'),(
    ''),'Open CASCADE STEP processor 7.6','FreeCAD','Unknown');
FILE_SCHEMA(('AUTOMOTIVE_DESIGN { 1 0 10303 214 1 1 1 1 }'));//表明数据段schema
ENDSEC;//段结束标志，上面三个实体在必须出现且按顺序出现，其余可选
DATA;//数据段开始
/*
 在ISO-10303-214 中搜索entity APPLICATION_PROTOCOL_DEFINITION 可以找到
 
 ENTITY application_protocol_definition;
   status : label;
   application_interpreted_model_schema_name : label;
   application_protocol_year : year_number;
   application : application_context; //引用entity application_context
 END_ENTITY;
 
 其余数据段类似查找
 */
#1 = APPLICATION_PROTOCOL_DEFINITION('international standard',
  'automotive_design',2000,#2); 
#2 = APPLICATION_CONTEXT(
  'core data for automotive mechanical design processes');
...
ENDSEC; //结束段
END-ISO-10303-21; //协议结束

```

对于step文件的解析目前比较知名的开源免费软件是OpenCasCade（LGPL授权），目前能支持AP214CD,AP214DIS,AP203,AP214IS,AP242DIS 5个版本。 对于使用OCC的开发而言可能一般只关注TopoDSToStep和 StepToTopo里面的图形转换，自行开发step引擎的话还是需要仔细阅读ISO-10303相关文档











<br/>
<br/>
<br/>

参考文献：
1.ISO 10303-1:1994
2.ISO 10303-21:2002 
3.ISO 10303-214-2003
4.https://blog.csdn.net/lyalong0616/article/details/90231001
5.https://www.capvidia.com/blog/best-step-file-to-use-ap203-vs-ap214-vs-ap242
6.https://dev.opencascade.org/doc/overview/html/occt_user_guides__step.html 