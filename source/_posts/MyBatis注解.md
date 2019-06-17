---
title: MyBatis相关注解
date: 2018-05-17 20:09:55
tags: [Java,SSH]
categories: 框架
toc: true
top: true
---
![](http://img.gangtieguo.cn/006tNbRwgy1fu551ai6pij30mm064q3r.jpg)

现接触MyBatic记录一些注解
<!--more-->
自动生成主键  

        可以使用 @Options 注解的 userGeneratedKeys 和 keyProperty 属性让数据库产生 auto_increment（自增长）列的值，然后将生成的值设置到输入参数对象的属性中。 
```java
@Insert("insert into students(name,sex,age)  values(#{name},#{sex},#{age}")  
@Options(useGeneratedKeys = true, keyProperty ="userId")  
int insertUser(User user);   
```
将自增的Id存入到userId属性中