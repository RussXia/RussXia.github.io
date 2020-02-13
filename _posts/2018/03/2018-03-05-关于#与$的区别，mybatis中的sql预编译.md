---
layout: blog
title: "关于#与$的区别，mybatis中的sql预编译"
catalog: true
tag: [Java,Mybatis,2018]
---
# 关于#与$的区别，mybatis中的sql预编译

## \'#{}\'与\'${}\'的区别

### 结论

'#{}'是经过预编译的，是安全的;

${}仅仅是一个string替换，存在sql注入的可能性。
**能使用#{}的时候尽量使用#{}**

### 二者的区别

在Mybatis的动SQL解析阶段，使用#{}和${}会有不同的表现:

#### '#{}'

'#{}'会解析成一个JDBC预编译语句(PreparedStatement)的参数标记符(占位符)。会有一个PreparedStatement类型检查和安全检查的过程

`select * from user where user_name = #{name};`

上面的sql会被解析为:

`select * from user where user_name = ?;`

#### ${}

${}只会做简单的字符串替换，在动态解析阶段就会替换掉${}变量

`select * from user where user_name = ${name};`

假如传递的参数为Tom,上面的sql会被解析为:

`select * from user where user_name = 'Tom';`

${}变量的替换阶段是在动态sql解析阶段，在预编译之前，sql中已不包含变量了。

#### '#{}'与'${}'方式的对比

+ #{}可以防止sql注入
+ mybatis默认对所有SQL预编译,相同的预编译sql可以重复利用。
+ 表名或者列名作为变量时，必须使用 ${}。(使用#{}会带上字符串`''`,导致sql语法错误,order by ${} !!!)

## PreparedStatement预编译

PreparedStatement有三个显著的优点:

+ 代码的可读性和可维护性.
+ PreparedStatement尽最大可能提高性能.
+ 最重要的一点是极大地提高了安全性(防sql注入)
