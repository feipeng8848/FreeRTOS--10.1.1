FreeRTOS 10.1.1 2018-11-03下载自官网
学习笔记
---
#一、代码规范
FreeRTOS 源代码遵守 MISRA (Motor Industry Software Reliability Association) 规范。

与 MISRA 标准有出入的地方如下：
-两个 API 函数具有两个出口点。之所以这样是为了效率。 
-使用标准 C 数据类型，而不是用 typedef 将其名称重定义。
-当建立一个任务时，代码会直接处理堆栈的栈顶和栈底地址。由于不同的平台的总线宽度不同，这就需要代码中对指针变量进行算术运算。因此，对指针变量的算术运算是不可避免的。
-trace 宏定义，默认情况下被定义为空，因此不会产生任何代码。

##命名约定(Naming Conventions)

RTOS内核与Demo程序源代码使用下面的约定：

####变量
char类型的变量以 c 为前缀 
short类型的变量以 s 为前缀 
long类型的变量以 l 为前缀 
float类型的变量以 f 为前缀 
double类型的变量以 d 为前缀 
枚举变量以 e 为前缀 
其他类型（如结构体）以 x 为前缀 
指针有一个额外的前缀 p , 例如short类型的指针前缀为 ps 
无符号类型的变量有一个额外的前缀 u , 例如无符号short类型的变量前缀为 us


####函数
文件内部函数以prv为前缀 
API函数以其返回值类型为前缀，按照前面对变量的定义 
函数的名字以其所在的文件名开头。如vTaskDelete函数在Task.c文件中定义 

####宏定义
宏名以所在的文件的文件名的一部分作为前缀（开头），并且用小写。 
比如, configUSE_PREEMPTION 在文件 FreeRTOSConifg.h 中. 
除了前缀，其余部分用大写，下划线来分隔单词。

####数据类型

基本数据类型可以直接使用，但是有如下的例外和规则：

char类型在每个平台都有其自身的定义方式。有些平台 char 等价于 signed char ，另一些则等价于 unsigned char，为此，要在代码中明确的使用 signed char 或 unsigned char 。直接使用 char类型是被禁止的。
不能直接使用 int 类型，要使用 short 和 long。
 float 和 double 没有在内核中使用，但是Demo 代码中有使用。

此外，有两种额外的类型要为每种平台定义。分别是：

portTickType

如果 configUSE_16_BIT_TICKS 被定义, 则 portTickType 被定义为无符号16bit 类型，否则为无符号 32 bit 类型。参考API文档中的 定制部分获取详细信息。


portBASE_TYPE
被定义为当前平台最佳的整形类型。例如，在一个 32 位的平台上， portBASE_TYPE 被定义为32 位的数据类型。在16位的平台上， portBASE_TYPE 则被定义为 16 位的数据类型。如果 portBASE_TYPE 被定义为 char 类型，则 必须为 signed char  类型，因为代码中用到这种类型作为一些函数的返回值类型，而返回值必须可以为负值以用来指示错误条件.

####编程风格
缩进
缩进使用 Tab . 一个 tab 等于 4 个空格.

####注释 
注释文字尽量不能超过 80 列，除非是用来描述一个参数。 
不采用 C++ 类型的注释（//）。 
---
二、源代码分析
1、list.c
