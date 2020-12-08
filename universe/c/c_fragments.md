---
title: C语言碎片知识
date: 2019-12-04
tags: c, c++
mathjax: true
---

## 读取字符串

- 读取单个字符
    * `getchar()`是`scanf("%c", c)`的简化版本，除了更简介无其他优势
    * `getche()`，没有缓冲区，输入一个字符后立即读取，不等待用户按回车
    * `getch()`，没有缓冲区，输入一个字符后立即读取，不等待用户按回车，区别于`getche`它**没有回显** 
- 读取字符串
    * `scanf("%s", str)`遇到空格后停止接收后面的字符串
    * `gets()`有缓冲区，可以读取空格，直到回车才会结束
    * `scanf("%[^\n]",str)`，可以读取带空格的字符串，直到回车才结束


## 函数指针

函数指针声明如下

```c
返回值类型(*指针变量名)(参数列表)
int(*func_ptr)(int, int) = & func; // &可以省略
```

上面声明了一个指向 *返回值为int，有两个int类型参数的函数* 的指针`func_ptr`，接下来就可以通过`func_ptr(int, int)`调用了


### 回调函数

函数指针变量作为某个函数的参数，即别人的函数执行时调用你实现的函数


## static

static有如下功能

- 1. 隐藏变量
    * 对于未加static的全局变量和函数具有全局可见性，可在另一个文件中`extern`关键字声明使用
- 2. 保持变量内容的持久
    * 存储在静态数据区的变量会在程序运行时就初始化，且也是唯一一次初始化。全局变量和static变量就是静态数据区的变量
    * 不同于全局变量，static可以控制变量的可见范围，也就是隐藏局部变量，但局部变量从储存是全局的
- 3. 初始化为0
    * 静态数据区中所有的字节默认都是0x00


## 宏

简单的讲，宏就是把一个缩略语(宏，macro)指定成任何一段文本。预处理时会用定义和文本把这个缩略语替换掉。

- 没有参数的宏
    * `#define 宏名称 文本`
- 带参数的宏
    * `#define 宏名称(参数列表) 文本`
    * **注意** 宏参数的替换也是文本的替换，因此不要吝啬括号
        ```c
        #define T(x, y) x*y
        T(1+1, 2) // 结果是1+1*2 = 3
        #define T(x, y) (x)*(y) // 这样才能得到4
        ```
- 可选参数的宏
    * `#define 宏名称(参数列表,...) 文本`
        + 省略号必须放在参数列表的后面，表示可选参数
    * 可选参数会用**连同分隔他们的逗号**打包在`__VA_ARGS__`中
- 字符串化运算符
    * 形参中使用`#`可以将宏中的实参转换用双引号`"`包裹成字符串
        + 如果实参有双引号`"`，则每个双引号前面会加反斜杠`\`
        + 同理反斜杠以后自动转意
        + **PS** 编译器会合并紧邻的字符串字面量(`"ab""c" -> "abc"`)
- **记号粘贴运算符**
    * `##`，会把左、右操作数组合在一起作为一个记号
        + `#define A(x) a_##x -> A(1) -> a_1`
        + 出现在`##`两边的空白符会连同`##`一起删除
        + 如果得到的结果是宏则预处理器继续进行宏替换，即 **记号粘贴完成后才进行宏展开**


## 条件编译

不同平台提供的api不同，根据平台编译代码是必须的

`#if`, `#elif`, `#else`, `#endif`都是预处理命令，这些操作都是在预处理阶段完成的，多余的代码以及宏都不会参与编译，保证正确性还缩小了体积。

`#ifdef`，意思是如果宏被定义过，`#ifndef`则是它的非

需要注意的是`#if`后面跟的是"整型常量表达式"，而`#ifdef`和`#ifndef`后面只能跟宏名


## 枚举enum

用符号名称代表常数，提高程序可读性。枚举常量是int类型的，因此任何int类型使用的地方都可以用它。定义枚举类型方法如下

```c
enum 枚举名 {枚举元素1, 枚举元素2,...};
```

**需要注意的是** 第一个枚举元素默认为0，后续元素在前一个元素上加1。

可以在声明时指定枚举元素的值，没指定值的元素会在前一个元素的值上加1

```c
enum H {A, B=3, C, D=7, E};
// A=0, B=3, C=4, D=7, E=8
```

### 定义枚举变量

枚举变量是值可以用枚举类型中定义的名称赋值(当然也可以用任意int或unsigned int赋值，但这么做就失去了可读性)，如`enum day = MON`。定义枚举变量有三种方式

```c
// 先定义枚举类型再定义枚举变量
enum DAY
{
    MON=1, TUE, WED, THU, FRI, SAT, SUN
}; 
enum DAY day;

// 先定义枚举类型同时定义枚举变量
enum DAY
{
    MON=1, TUE, WED, THU, FRI, SAT, SUN
} day; 

// 省略枚举类型名，直接定义枚举变量
enum 
{
    MON=1, TUE, WED, THU, FRI, SAT, SUN
} day; 
```


## goto

向后跳转

```c
goto label;
..       |
..       |
label:  <
    statement;
```

向前跳转

```c
label:         <-|
    statement    |
..               |
..               |
goto label;     -
    statement;
```


