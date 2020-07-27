---
title: vim实用技巧
date: 2020-2-26
tags: vim
---

# Vim实用技巧


## 杂项
| 操作              | 描述                                             |
|-------------------|--------------------------------------------------|
| o                 | 切换高亮中光标所在的端点                         |
| .                 | 重复执行上一步操作                               |
| %                 | 它代表当前文件中的所有行                         |
| %                 | 可以在一组开, 闭括号间跳转                       |
| m{mark}           | 当前位置标记为{mark}以便以后跳转                 |
| \`{mark}          | 跳到{mark}标记处                                 |
| qa                | 录制宏到寄存器a                                  |
| qA                | 往寄存器a中追加操作                              |
| i/d{              | i for inside, a for around, 就可以用di/da等操作  |
| =i{               | =格式化?(缩进), inside {}                        |
| :earlier/later 1m | 回到1min前/后                                    |
| c-z               | 挂起vim,命令行下:$ fg可以恢复,jobs查看挂起的进程 |
| c-a/c-x           | 数字加/减1                                       |


## 查找

| 操作     | 描述                   |
|----------|------------------------|
| *        | 向下查找光标停靠的词语 |
| #        | 向上查找               |
| ;        | 重复上一次查找         |
| ,        | 反转方向重复上一次查找 |
| \ze和\ze | 匹配界定符, 匹配边界   |

### 按正则表达式查找
| 操作    | 描述                                      |
|---------|-------------------------------------------|
| \v      | 使用正则的特殊符号规则                    |
| \V      | 原意查找, 即.啊什么的不用反斜转意         |
| \w      | 用来匹配单词类字符, 包括字母,数字及符号   |
| \W      | 匹配单词类以外的所有字符                  |
| \\{num}  | 引用被没对()捕获的子匹配 \0会引用整个匹配 |
| <和>    | \v模式中<和>用来匹配边界                  |


## 操作符 + 动作 = 操作
### 操作符
| 操作 | 描述         |
|------|--------------|
| c    | 修改         |
| d    | 删除         |
| y    | 复制到寄存器 |
| g~   | 反转大小写   |
| gu   | 反转为小写   |
| gU   | 反转为大写   |
| >    | 增加缩进     |
| <    | 减小缩进     |
| =    | 自动缩进     |
| !    | 使用外部程序 |

### 动作
| 操作 | 描述                                 |
|------|--------------------------------------|
| w    | 向后一个单词                         |
| W    | 当前字串                             |
| aw   | a word, 光标停靠的整个单词及一个空格 |
| ab   | 一对圆括号                           |
| aB   | 一对花括号                           |
| t    | to 到                                |
| i"   | inside 在""中                        |
| it   | inside tag <h1></h1>                 |
| p    | 段落                                 |
| s    | 句子                                 |
| @a   | 宏a                                  |

### 例子
| 操作 | 描述                 |
|------|----------------------|
| caw  | 修改当期单词         |
| dta  | 删除当前位置到字母啊 |
以此类推


## Ctrl+x补全

| 操作        | 描述                          |
|-------------|-------------------------------|
| c-x,c-r     | insert from a register        |
| c-x,c-a     | last inserted text            |
| c-x,c-]     | tag completion                |
| c-x,c-f     | filename completion           |
| c-x,c-p/c-n | context-aware word completion |
| c-x,l       | context-aware line completion |


## 寄存器
| 操作            | 描述                                   |
|-----------------|----------------------------------------|
| "{register}     | 制定寄存器{register}                   |
| ""              | 无名寄存器, 没指定寄存器情况下缺省使用 |
| "0              | 复制专用寄存器                         |
| "\_             | 黑洞寄存器, 有去无回                   |
| "+              | X11剪贴板                              |
| \<C-r>{register}| 在在插入模式下插入寄存器内容           |
| :reg a          | 查看寄存器a                            |
等等


## Shell命令
| 操作                 | 描述                                  |
|----------------------|---------------------------------------|
| %                    | 命令行中的%会展开成当前文件的完整路径 |
| :shell               | 启动一个shell(输入exit返回vim)        |
| :!{cmd}              | 在shell中执行{cmd}                    |
| :read !{cmd}         | 把{cmd}的标准输出插入到光标下方       |
| :[range]write !{cmd} | 以[range]作为{cmd}的标准输入          |


## 分屏操作
| 操作      | 描述                   |
|-----------|------------------------|
| \<C-w>=    | 使得所有窗口等宽, 登高 |
| \<C-w>\_    | 最大化活动窗口的高度   |
| \<C-w>\|     | 最大化活动窗口的宽度 |
| \<C-w>方向 | 移动光标所在的窗口     |


## 地址
| 操作 | 描述                   |
|------|------------------------|
| 0    | 虚拟行, 位于第一行上方 |
| 1    | 文件第一行             |
| $    | 最后一行               |
| .    | 光标所在行             |
| 'm   | 包含位置标记m的行      |
| '<   | 高亮选取的起始行       |
| '>   | 高亮选取的结束行       |
| %    | 整个文件               |


## 宏

录制宏其实是把操作过程放入了一个寄存器中，如`qq`就是用寄存器q来录制宏。用`:reg q`可以查看内容。

- `qq`：用寄存器q录制宏
- `qQ`:往寄存器q中追加操作
- `:put q`：把寄存器q的内容粘贴到文档，以便修改
- `:d q`：复制回寄存器