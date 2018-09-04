---
title: Scala基本使用-杂记
date: 2018年08月06日 22时15分52秒
tags: [Scala]
categories: 语言
toc: true
---

[TOC]

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fukx028pz0j313t0op400.jpg)

Var和val

var 修饰的变量可改变，val 修饰的变量不可改变；但真的如此吗？事实上，var 修饰的对象引用可以改变，val 修饰的则不可改变，但对象的状态却是可以改变的。

## 定义方法
```scala
def m1(x:Int,y:Int):Int=x*y
```

<!-- more -->

## 函数的定义=>

```scala
val func:Int =>String={x=x.toString}
也可以写成
val func1=(x:Int)=>x.toString
```


## 函数和方法的区别
```scala
定义一个方法
def m2(f:(Int,Int)=>Int) =f(2,6)
定义一个函数
val f2 =(x:Int,y:Int) =>x-y
调用
f2(m2)
```


## 神奇的下划线
将方法转换成函数

```scala
def m2(x:Int,y:Int):Int = x+y
val f2(a:Int,y:Int)=>x+y
val f2 = m2 _
```




# 数组、映射、元组、集合
## 数组
```scala
val arr = new Array[Int](10) //创建数组，无值
val arr1 = Array(1,2,3,4,5)  //直接实例化
for(e<- arr) **yield** e*2; 用yield关键字可以生成一个新的数组 生成的类型和循环的类型是一样的
```


## map方法
map方法是将每一个元素拿来操作
arr.map(_*2) map方法更好用  map是排序？

```scala
val a = Array(1,2,3,4,5)
a.map((x:Int)=>x*10)//匿名函数
```


由于知道a中的数据类型 可以将Int省略
a.map(x=>x*10) 
还可以用_占位符，进行进一步的省略
a.map(_ * 10)


所有偶数取出来然后再乘以10
```scala
arr.filter((x:Int)=>x%2==0)
arr.filter(x=>x%2 == 0)
arr.filter(_%2==0)
arr.filter(_%2==0).map(_*10)
val p =  println _
```



可以将方法转换成函数用利用 _
### trait 关键字
## 数组的常用函数
在超类
TraversableLike中定义了这些函数
```scala
val arr = Array(1,2,3,4,5,6)
arr.sum 总数
arr.sorted 排序
arr.sorted.reverse 逆序
arr.sortBy(x=>x)按照本身来排序
arr.sortWith(_>_)从大到小排序 < 从小到大排序
arr.sortWith((x,y))
## 
```

映射 类比java中的map
```scala
val m = Map("a"->1,"b"->2)
m("a") 取值 
```


里边的值不能更改
如果是mulitble的包就可以更改了
```scala
m.getOrElse("c",0) 找c 如果没有c就创建0
```


### apply方法
##  元组

val t = (1,"spark",2.0); 值类型都不一定

m.+=(("c",1))  与 m+=("m"->1) 写法相同
val t,(x,y,z) = ("a",1,2.0) 
t是变量，x y z 是键 对应三个值，取值的时候 直接用x就可以取值

### 将对偶的转成映射
```scala
val arr  = Array(("a",1),("b",2))
arr.toMap 就可以转成映射
```


### 拉链操作
```scala
val a = Array("a","b","c")
val b = Array(1,2,3);
a.zip(b) 
```


将其拉在一起变成数组，元素为元组
# 集合
序列Seq,集Set,映射Map
集合分为可变和不可变的 mutable 和 immutable
注意和val的对比
将0插入来lst前面生成一个新的集合

```scala
val lst2 = 1 :: lst1; 
val lst3 = lst1.::(0)
val lst4 = 0 +: lst1
val lst5 = lst1.+:(0)
```


将一个元素添加到lst后面产生一个新的集合

```scala
val lst6 = lst1:+3
val lst0 = List(4,5,6)
```


将2个list合并成一个新的List

```scala
val lst7 = lst1 ++ lst0
```


将lst0插入到lst前面生成一个新的集合

```scala
val lst8 = lst1 ++: lst0
```



## foreach
foreach是将其值取出来，不会新生成一个集合
map将值取出来会新生成一个集合
## Map
Map本身不支持排序
但是有toList方法 无括号，可以支持排序
## List
```scala
val list = List(1,2,3,4,5,6);
```


list.par转成并行化集合
list.par.reduce(_+_) 将其放在多个reduce中执行，数据量大的时候将会变得很快
reduce不是并行集合的话，就是调用的底层
reduceLeft(不能并行了)
## fold可以指定初始值
```scala
list.par.fold(0)(_+_)
```


## aggregate 聚合
需要传两个函数，第一个函数是对元素进行操作，第二函数是对局部操作的结果进行操作

```scala
List（List(1,7,9,8),List(0,1,2,3)）
list.aggregate(0)(_+_.sum,_+_)
```



union 并集，intersect交集，diff差集


## flatten 将数据压缩
```scala
List（List(1,7,9,8),List(0,1,2,3)）
listAll.flatten
List(1,7,9,8,0,1,2,3) 产生新的集合
```



# 对象

主构造器里面的所有方法都会被执行
## 单例对象
所有的object都是一个单例（把class替换成object）
不要new，直接等于类名就是调用的一个**单例对象**

```scala
object Dog{
  def main(){
  val d = Dog
  }
}
```


## 伴生对象
就是对象名和类名一样，并且在一个scala文件中
可以和类**互相访问私有属性**
scala中返回的就是unit就是返回的一个括号
### apply方法
```scala
Object Dog{
 def apply():Unit = {
  print ();
 }
 def apply(name:String):Unit = {
  print(name)
 }
 def main(){
 //会调用第一个无参数的apply方法
 val d1 = Dog()
 //会调用第有参数的apply方法
 val d2 = Dog("haha");
 }
}
```

## 应用程序对象
没有什么实际作用
## 构造函数
用this关键字定义辅助构造器
```scala
def this(name:String,age:Int,gender:String){
//每个服务构造器必须以主构造器或者其他的辅助构造器的调用开始
//主构造器就是类名上直接填写参数
}
```

函数与方法的互换， 神奇的下划线
要想传到map里面，必须得是函数

```scala
def  fangfa方法
val  func 函数
val arr = Array(1,2,3,4,5,6)
arr.map(func(5))
arr.map(func())
val m = fangfa _  方法func转成函数
fangfa()  也可以转函数

def m(x:Int) = (y:int)=>x*y
```



## 柯里化的两种表达方式
柯里化是主要是通过类型类匹配的

```scala
def m1(x:Int) = (y:Int)=>x*y
def m2(x:Int)(y:Int)=>x*y
```


柯里化会先执行一部分，返回一个函数
```scala
 def multi= (x:Int) =>{
    x*x
  }

  def main(args: Array[String]): Unit = {
    val arr = Array(1,2,3,5,4)
    val a1 = multi(10)
    println(s"平方式：${a1}")
    //按照规则 map只能参数只能是函数，multi是一个方法，但是在柯里化的时候，会先返回一个中间结果是函数
    val a2 = arr.map(multi)
  }

```
## 继承 代理 装饰 之间的区别
继承是类的增强

代理是对实例，方法的增强

装饰也是对方法的增强

implicit def 隐式的，隐式转化的包在predef中
# 泛型

```scala
<? extends clazz> 传入的数据是clazz的子类   
<? super clazz> 传入的数据是clazz的父类   
```



## > < >= <=

以上操作符，在scala中都是方法





### 视图定界 view bound <%

scala泛型

```scala
class Person[T] { 
def chooser[T <: Comparable[T]](firit: T, second: T): T = { 
	first 
	} 
} 
```


隐式转换：我自己的隐式上下文

```scala
object MyPredef{ 
implicit 函数 
implicit 值 
} 
```


viewbound要求传入一个隐式转换函数

```scala
class Chooser[T <% Ordered[T]] { 
def bigger(first: T, second: T) : T = { 
	if(first > second) first else second 
	} 
} 


class Chooser[T] { 
def bigger(first: T, second: T)(implicit ord: T => Ordered[T]) : T = { 
	if(first > second) first else second 
	} 
} 
```


contextbound要求传入一个隐式转换值

```scala
class Chooser[T: Ordering] { 
def bigger(first: T, second: T) : T = { 
val ord = implicitly[Ordering[T]] 
if(ord.gt(first, second)) first else second 
} 
} 
class Chooser[T] { 
def bigger(first: T, second: T)(implicit ord : Ordering[T]) : T = { 
if(ord.gt(first, second)) first else second 
} 
} 
```


[+T] 
[-T]

相当于传入了一个隐式转换的函数
一定要传入一个隐式转换函数

```scala
class Chooser [t <% Order[T]]{
  def choose(first T,second T) :T ={
  //val ord = implicitly[Ordering[T]]
  // if(ord.gt(first,second))) first else second;
   if(first.compare(second) > 0) first else second;
  }
}
```

```scala
implict object girlOrdering extends Ordering[girl]{
override def compare(x:girl,y:girl):Int = {

}
}

== 和这个实现的效果是一样的,只是取了一个名字
implicit val girlOrder = new Ordering[Girl]{

}

```




### 上下文定界 : content bound
相当于传入了一个隐式转换的值

关于实体类Predef的关系