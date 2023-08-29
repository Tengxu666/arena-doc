---
date: 2018-12-06
title: Docsy的简单文档
linkTitle: 宣布Docsy
description: >
    Docsy Hugo主题让项目维护者和贡献者专注于内容，
    而不是从头开始重新设计网站基础设施
author: tengxu.liu
---

**这是一篇典型的包含图片的博客文章。**

前端内容指定博客文章的日期、标题、将显示在博客登录页面上的简短描述以及作者

## 包括图像

这是一张图片（`featured-sunset1-get.png`），其中包括一条署名和一个标题。

{{< imgproc sunset Fill "600x300" >}}
获取并缩放即将发布的Hugo 0.43中的图像。
{{< /imgproc >}}

要想明白 Go 语言中的指针，需要知道以下概念

### 变量

对于变量，可以认为就是给内存中一块区域指定一个名称。我们可以通过这个名称直接访问这块内存。既然是变量，意思也就是说这块儿内存里的内容，也就是变量值，是可以改变的。我们可以通过使用这个变量名，通过某些方式对这块儿内存中的值进行修改。这也是区分于变量和常量的关键点。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/27548527/1691115571463-f5cfb6ed-d25c-4c61-9d6c-ab1793b23848.png#averageHue=%23fdfcfc&clientId=ue58ec17d-c322-4&from=paste&height=515&id=u68c143ea&originHeight=337&originWidth=531&originalType=binary&ratio=2&rotation=0&showTitle=false&size=67482&status=done&style=none&taskId=u7fff5003-099d-41f1-b638-320c57bc3ec&title=&width=812)
通过上图，我们基本上可以了解，Go语言中的变量基本上有三个元素构成：

- 变量名（var1、var2）
- 内存（每快内存都有一个**内存地址**）
- 变量的数据类型，标明了这块内存只能存放哪种类型的数据

> 变量名也应该是需要在内存中存储的，不过对于变量名怎么映射到内存，并且变量名又是如何进行访问的，这些都有底层来自动实现。不同的编程语言都有自己的内存模型，这里我们只关心用户层面变量的访问


### 指针

我们知道变量是用来存储数据的，变量的本质是给存储数据的内存地址起了一个好记的别名。比如我们定义了一个变量`a := 10`，这个时候可以直接通过a这个变量来读取内存中保存的10这个值。在计算机底层a这个变量其实对应了一个内存地址。
指针也是一个变量，但它是一种特殊的变量，它存储的数据不是一个普通的值，而是另一个变量的内存地址。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/27548527/1691116027864-2392cebd-4920-428c-96b6-b60b9cad75e1.png#averageHue=%23f4f2ea&clientId=ue58ec17d-c322-4&from=paste&height=462&id=u1d07e113&originHeight=444&originWidth=750&originalType=binary&ratio=2&rotation=0&showTitle=false&size=158656&status=done&style=none&taskId=u3985f9d1-d7a2-42c8-9e7d-c28e450079c&title=&width=780)

### 指针地址和指针类型

每个变量在运行时都拥有一个地址，这个地址代表变量在内存中的位置。
Go 语言中使用 `&` 字符放在变量前面对变量进行取地址操作。
Go语言中的值类型 `int、float、bool、string、array、struct`都有对应的指针类型，如：
`*int、*int64、*string` 等
取变量指针的语法如下：

```go
ptr := &v
```

其中：

- `v`：代表被取地址的变量，类型为T
- `ptr`：用于接收地址的变量，ptr的类型就为T，被称做T的指针类型。代表指针

举个例子：

![image.png](https://cdn.nlark.com/yuque/0/2023/png/27548527/1691116408360-01c35c46-436b-4d26-8135-31bb4620b63b.png#averageHue=%23f5f3f3&clientId=ue58ec17d-c322-4&from=paste&height=521&id=u6d872bcd&originHeight=487&originWidth=750&originalType=binary&ratio=2&rotation=0&showTitle=false&size=90628&status=done&style=none&taskId=uda9263b9-e796-42bc-a6a8-b112591aaa6&title=&width=802)

### 指针取值

在对普通变量进行`&`操作符取地址后，会获得这个变量指针，然后可以对指针使用`*`操作，也就是指针取值。

```go
func main() {
	var (
		a = 10
		b = &a
	)
	println(a, b, *b)
	// 修改 b 的值
	*b = 20
	println(a, b, *b)
}

// 输出结果
// 10 0x14000046760 10
// 20 0x14000046760 20
```

### new 和 make 函数

需要注意的是，指针必须在创建内存后才可以使用。

```go
var a *int
*a = 100
fmt.Println(a)
```

执行上面的代码会引发panic，为什么呢？

> 在Go语言中对于引用类型的变量，我们在使用的时候不仅要声明它，还要为它分配内存空间，否则我们的值就没办法存储。而对于值类型的声明不需要分配内存空间，是因为它们在声明的时候已经默认分配好了内存空间。
> 要分配内存，就需要new和make。Go 语言中new和make是内建的两个函数，主要用来分配内存。

#### 区别

1. 两者都是用来做内存分配的
2. make只能用于slice、map以及channel的初始化，返回的还是这三个引用类型的本身
3. new用于类型的内存分配，并且内存赌赢的值为类型的零值，返回的是指向类型的指针

#### 用法实例

```go
// 使用new关键字创建指针
 := new(int)
fmt.Printf("%T", a)
fmt.Println(*a)

// 输出
*int 0


// 使用map关键字创建指针
var b map[string]int
b = make(map[string]int, 10)
b["a"] = 100
fmt.Println(b)

// 输出
map[a:100]
```

### 


