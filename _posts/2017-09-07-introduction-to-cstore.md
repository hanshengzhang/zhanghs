---
layout: post
category: tech
title: 列存储数据库C-Store介绍
---
<script src="https://code.jquery.com/jquery-1.12.4.js" type="text/javascript"></script>
<script src="https://cdn.datatables.net/1.10.16/js/jquery.dataTables.min.js" type="text/javascript"></script>
<script type="text/javascript">
var empData = [
  ["Bob", 25, "Math", "10K"],
  ["Bill", 27, "EECS", "50K"],
  ["Jill", 24, "Biology", "80K"],
  ["Alice", 23, "Art", "20K"],
  ["Jason", 28, "Physics", "30K"],
  ["John", 27, "Chemistry", "40K"]
];
var deptData = [
  ["EECS", 2], 
  ["Math", 5],
  ["Physics", 7],
  ["Chemistry", 10],
  ["Biology", 16],
  ["Art", 24]
];

function tableShow(element, data, columnNames) {
  element.DataTable({
    paging:false,
    data: data,
    bInfo : false,
    searching: false,
    sorting: false,
    columns: columnNames.map(function(x) {return {"title": x}})
  })
}

$(document).ready(function() {
  
  tableShow($('#logical_emp'), empData, ["Name", "Age", "Dept", "Salary"]);
  
  tableShow($('#logical_dept'), deptData, ["Name", "Floor"]);
  
  tableShow(
    $('#emp_proj1'), 
    empData.map(function(x){return [x[0], x[1]];}).sort(function(x, y){return x[1]-y[1];}), 
    ["Name", "Age"]
  );
  
  var deptMap = deptData.reduce(function(map, obj) {
    map[obj[0]] = obj[1];
    return map;
  }, {});
  var proj2 = empData.map(function(x) {return [x[2], x[1], deptMap[x[2]]];}).sort(function(x, y){ return x[2]-y[2];});
  tableShow($('#emp_proj2'), proj2, ["Dept", "Age", "DEPT.Floor"])
  
  tableShow(
    $('#emp_proj3'), 
    empData.map(function(x){return [x[0], x[3]];}).sort(function(x, y){return parseInt(x[1]) - parseInt(y[1]);}), 
    ["Name", "Salary"]
  );
});
</script>

C-Store是一个为了快速查询而设计的关系型数据库，它的论文发表于2005年的VLDB。
为了达到更快的查询性能，C-Store按列存储数据，同一个表中的不同列可能被存在不同的、可能有重叠的Projection中。
虽然C-Store没有提出什么新的技术，但它是第一次将与列存储相关的技术整合到一起，成为一个真正理论上可用的系统，所以其设计和系统架构仍被广泛借鉴。
HPE Vertica这款商业存储系统，就是基于C-Store设计并实现的。

<!--snapshot-->

本文主要基于其VLDB2005的论文，介绍C-Store中的相关概念与设计。

# C-Store数据模型

由于C-Store是关系型数据库，所以在C-Store中，表的Schema也和普通关系型数据库一致。
例如，有如下两张表：一张是员工表，EMP，其中有姓名、年龄、部门和工资这几列；另一张表是部门表，DEPT，有名称和楼层这两列。

<table id="logical_emp" style="float:left"><caption>员工表EMP</caption></table>
<table id="logical_dept" style="float:left;margin-left:8em"><caption>部门表DEPT</caption></table>
<div style="clear:both"> </div>

这两张表是C-Store中的逻辑表，虽然在表中我们标注了每一行，但是在C-Store中，逻辑表是不存放数据的，只是用来提供查询的Schema的，C-Store中的数据是存放在关联到逻辑表的几个Projection中。
C-Store的Projection有如下特点：1. 存储了逻辑表中的一列或多列；2. 按照其中的一列或者多列的组合进行排序；3.可能会存放其他逻辑表中的列。例如，EMP表可能有如下三个Projection。

<table id="emp_proj1" style="float:left"><caption>Projection1</caption></table>
<table id="emp_proj2" style="float:left;margin-left:8em"><caption>Projection2</caption></table>
<table id="emp_proj3" style="float:left;margin-left:8em"><caption>Projection3</caption></table>
<div style="clear:both"> </div>

上面的Projection1有姓名和年龄两列，按照年龄排序。Projection2中有部门、年龄以及部门对应的楼层，按照楼层进行排序。Projection3中有姓名和工资两列，按照工资进行排序。
Projection按照一列或者多列进行排序，能够更高效的压缩和更快速的查询。
查询时，C-Store会先选择一组相对合适的Projection，然后进行计算。
为了支持同一个表能够放在不同的节点上，C-Store中的一个Projection按照排序列的取值范围，划分为不同的Segment，Segment支持备份。

有时候，查询可能需要同一个逻辑表中的多个Projection才能完成，例如如果上述例子中，想要同时查询姓名和部门，这就需要Projection1和Projection2。
所以C-Store支持将多个Projection的列构造成一条记录，这个操作一般叫Tuple Construction。
C-Store实现Tuple Construction的逻辑比较简单，就是记录了一下每一个值的行号，然后在两个Projection之间记录一个叫Join Index的表，告诉这两个Projection要怎么去做Join。

# C-Store架构

## 基本架构

<svg width="800" height="95">
  <rect width="250" height="25" style="fill:rgb(255,255,255);stroke-width:2;stroke:rgb(0,0,0)" />
   <text x="20" y="17" fill="black">Writeable Store (WS)</text>
   <rect width="250" height="25" y="70" style="fill:rgb(255,255,255);stroke-width:2;stroke:rgb(0,0,0)" />
   <text x="20" y="87" fill="black">Read-Optimized Store (RS)</text>
   <line x1="125" y1="25" x2="125" y2="70" style="stroke:black;stroke-width:2"> </line>
   <polygon points="120,64 130,64 125,70" style="fill:black;stroke:black;stroke-width:1"></polygon>
   <text x="128" y="50" fill="black">Tuple Mover</text>
</svg>


C-Store的基本架构如上图所示，有两种类型的节点，一种是为了读优化的并且是只读的(RS)，另一种是可以写的(WS)。
插入、更新、删除等操作都发动到WS，而查询会发到RS和WS，并合并二者的结果。
数据周期性的通过一个叫Tuple Mover的组建从WS移动到RS。

**RS**是为了读而进行优化的，主要是对不同的列采用了不同的编码方式。
针对某一列，C-Store根据它是否是Projection的排序列，以及该列的取值个数，来决定采取何种编码方式。
具体说来，有如下几种：

- 是排序列，并且取值的个数比较少，例如1,1,2,2,2,2,2,2,2,2,3,3,3,3,3。
  C-Store会将这中列表示成一个三元组(v,f,n)的集合，v表示值，f表示该值的起始位置，n是该值一共的个数。
  这里的例子可以表示为：(1,1,2), (2,3,8),(3,11,5)。
- 不是排序的列，取值的个数比较少，例如0,0,1,1,2,1,0,2。对于这种列，采用Bitmap编码方式：(0, 11000010), (1, 00110100), (2,00001001)。
- 是排序列，取值个数比较多，例如1,4,7,7,8,12。这种情况下按照块进行编码，每块内记录第一个值，其他的值都记和前面的差值，(1,3,3,0,1,4)。
- 不是排序列，取值的个数比较多，在这种情况下，直接记录值，不进行编码。

**WS**是为了支持写的，WS中的每一列都用一个从行号到取值的B+树表示，对排序列，还记录一个取值到行号的B+树，以方便查询。

**Tuple Mover**周期性的把WS中的数据移动到RS中。
C-Store中的更新是通过一个删除加一个插入实现的，所以Tuple Mover的主要工作是载入RS中的数据，删掉在WS中被删除的，然后添加WS中新的数据。

## 事务支持

C-Store期望大部分的事务都是只读的，只有商量的支持更新的事务。
为了避免只读的事务有任何锁操作，C-Store使用了Snapshot Isolation的策略。
C-Store周期性的对已经提交的事务进行Snapshot，而只读的Transaction只能查询这些Snapshot而不是最新提交的结果。
对于少量的可以更新的事务，C-Store采用了传统的2PC。

## 查询执行

C-Store的查询执行和传统的Sql操作没有什么不同，主要是多了一个根据Bitmap来Filter数据的操作。
C-Store的查询优化最重要的一步是选出一组Projection。

# 总结

我认为，C-Store对分析型数据库而言，有着重要的意义。
在C-Store以前，大家会讨论不同的数据序列适合什么样的编码，会思考如何通过Data Mirror实现查询加速，会考虑到分布式数据存储中的事务支持，但是是C-Store，将这些技术糅合在一起，设计出一个分析型的列存储关系型数据库。
当然，从细节设计和实现上来讲，C-Store不可能说是尽善尽美，这些很多在商业产品HPE Vertica中都有改善，但我认为C-Store跨出了重要的一步，这一步不那么完美， 似乎也并不重要。
