---
title: C++程序员学Golang Part.1
description: >-
  
author: Dante.Cloud
date: 2025-07-06 00:00:00 +0800
tags: [Golang, C++]
---

职业生涯主要工作语言是C++，有过不太多的Golang经验。最近为了面试对go的语法和使用做了全面的学习，把要点记录下来，跟C++做对比加强记忆，也为有类似需求的朋友提供帮助。   
这是系列文章的第一篇，先介绍些基础知识。

## 变量定义
最明显的不同就是变量定义时的格式，Golang中的类型名都是写在变量名后面的：   
```var i int32 = 0```   
当然，也可以不写类型名，由编译器自己识别：   
```i := 0```

## 类型转换
Golang不允许隐式的类型转换，类似于：   
```go
var a int = 10
var b int64 = a
```   
的语句被视为错误，必须有明确的类型转换语法，且语法与C++也不同：   
```go
b = int64(a) 
```

## 数组
Golang中的数组被视为**原生类型**，例如数组同样跟其他基本类型一样也是有指针的：   
```go
arr := [5]int{1, 2, 3, 4, 5}
parr := &arr
fmt.Println("Array:", *parr)
```

## 切片
切片（slice）是对数组一个连续片段的引用，所以切片是一个引用类型。
切片的定义使得它跟数组的关系令人有些疑惑，但实际上底层数组并不会对切片的使用产生限制：
```go
arr := []int{1, 2, 3, 4, 5}
sli := arr[1:3]
fmt.Println("Slice:", sli)

sli = append(sli, 6) // 未超出原数组范围，仍然引用原数组
fmt.Println("Extended Slice:", sli)
fmt.Println("Original Slice after extending:", arr)

sli = append(sli, 7, 8, 9) // 超出原数组范围，需引用新的数组
fmt.Println("Further Extended Slice:", sli)
fmt.Println("Original Slice after further extending:", arr)
```
输出：
```
lice: [2 3]
Extended Slice: [2 3 6]
Original Slice after extending: [1 2 3 6 5]
Further Extended Slice: [2 3 6 7 8 9]
Original Slice after further extending: [1 2 3 6 5]
```
在切片体积超出原引用数组的情况下，就会隐式地引用新的数组，所以切片的使用与C++的vector是类似的。

## 字符串
不同于C++的字符串实质上就是“字节串”，golang的字符串类型string原生支持utf-8，每个“字符”占1～4个字节。   
string的字符类型为**rune**，是int32的别名（type rune = int32），用于表示Unicode码点，而不像C++字符串的基本单位是char，也不会以'\0'结尾。
```go
s := "Hello, 世界"
r := []rune(sts)
for i, r := range r {
    fmt.Printf("索引 %d: 字符 %c (码点 %U)\n", i, r, r)
}
```
输出
```
索引 0: 字符 H (码点 U+0048)
索引 1: 字符 e (码点 U+0065)
索引 2: 字符 l (码点 U+006C)
索引 3: 字符 l (码点 U+006C)
索引 4: 字符 o (码点 U+006F)
索引 5: 字符 , (码点 U+002C)
索引 6: 字符   (码点 U+0020)
索引 7: 字符 世 (码点 U+4E16)
索引 10: 字符 界 (码点 U+754C)
```
也可以把字符串解释为字节串：
```go
s := "Hello, 世界"
bs := []byte(s)
for i, b := range bs {
	fmt.Printf("Index %d: Byte %d \n", i, b)
}
```
输出：
```
Index 0: Byte 72 
Index 1: Byte 101 
Index 2: Byte 108 
Index 3: Byte 108 
Index 4: Byte 111 
Index 5: Byte 44 
Index 6: Byte 32 
Index 7: Byte 228 
Index 8: Byte 184 
Index 9: Byte 150 
Index 10: Byte 231 
Index 11: Byte 149 
Index 12: Byte 140 
```
strings包中有丰富的工具用于处理字符串，举几个简单的例子
```go
s := "hello world"
fmt.Println(strings.Contains(s, "world"))   // 是否包含子串
fmt.Println(strings.Index(s, "l"))        // 子串首先出现的索引位置
fmt.Println(strings.Count(s, "l"))        // 子串出现次数
```
其他的更复杂的工具函数还有很多，可以按需查找。

## map
C++有map和unordered_map，分别基于红黑树和链式哈希表实现；Golang的map基于链式哈希表实现，所以Golang的map不能像C++的map一样用一个迭代器来进行元素的有序遍历。   
跟C++一样也可以用下标来访问元素，但不同的是会有2个返回值，value和标识下标key是否存在的bool值。
```go
m := map[int]int{1: 10, 2: 20, 3: 30}
v, ok := m[2]
if ok {
	fmt.Println("Key 2 exists with value:", v)
} else {
	fmt.Println("Key 2 does not exist")
}
```


# 函数
Golang允许函数有多个返回值：   
```go
func fn(int) (int, int, int) {
    return 1, 2, 3
}
```
这点比C++方便很多，C++如果想达成这样的效果只能通过多个入参指针修改外部变量值的方式。   
函数的返回值可以当成几个并列的变量，甚至可以将一个函数的多个返回值作为另一个函数的入参：
```go
func fn1(int) (int, int) {
    return 1, 2
}

func fn2(int, int) (int, int) {
    return 3, 4
}

i1, i2 := fn1(1)
fn2(fn1(1))
```
还有注意，Golang的函数是不能重载的。

## “类”
Golang中的struct类似于C++的class，成员变量名大写类似于public，小写类似于private：
```go
type Person struct {
    name string  // 私有字段
    Age  int     // 公开字段
}
```
struct也可以有自己的方法：
```go
func (p Person) SetName(name string) {
    p.name = name
}
```
接受者也可以是类型指针：
```go
func (p *Person) SetName(name string) {
    p.name = name
}
```
实际调用的时候可以用struct实例调用也可以用指针调用
```go
person := Person{Age: 30}
person.SetName("王小明")
```
或
```go
person := &Person{Age: 30}
person.SetName("王小明")
```
类似于C++的继承功能，则需要内嵌来实现
```go
type Student struct {
    Person // 嵌入Person结构体
    No  int
}

student := Student{}
student.SetName("李四")
```
## make和new
都是申请空间，make限定使用于内置3种类型slice、map和channel：
```go
slice := make([]int, 0, 100)
hash := make(map[int]bool, 10)
ch := make(chan int, 5)
```
new则可用于任意类型：
```go
person := new Person{Age: 30}
```
或者不显式用new
```go
person := &Person{Age: 30}
```

无论是make还是new，都不会像C++的new一样确定性地把空间申请在堆上。编译器会分析变量的生命周期和作用域，若变量超出当前函数的作用域（如返回给调用者），则分配到堆上；否则可能到栈上。
```go
func createSlice() []int {
    s := make([]int, 3)  // s 逃逸到堆，因返回给调用者
    return s
}

func main() {
    s := createSlice()  // s 在堆上
    // ...
}
```
```go
func main() {
    s := make([]int, 3)  // s 分配到栈，因未逃逸出 main 函数
    // ...
}
```
