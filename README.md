FreeRTOS 10.1.1 2018-11-03下载自官网
学习笔记

---

# 一、代码规范
FreeRTOS 源代码遵守 MISRA (Motor Industry Software Reliability Association) 规范。

与 MISRA 标准有出入的地方如下：
-两个 API 函数具有两个出口点。之所以这样是为了效率。 
-使用标准 C 数据类型，而不是用 typedef 将其名称重定义。
-当建立一个任务时，代码会直接处理堆栈的栈顶和栈底地址。由于不同的平台的总线宽度不同，这就需要代码中对指针变量进行算术运算。因此，对指针变量的算术运算是不可避免的。
-trace 宏定义，默认情况下被定义为空，因此不会产生任何代码。

## 命名约定(Naming Conventions)

RTOS内核与Demo程序源代码使用下面的约定：

#### 变量
char类型的变量以 c 为前缀
short类型的变量以 s 为前缀 
long类型的变量以 l 为前缀 
float类型的变量以 f 为前缀 
double类型的变量以 d 为前缀 
枚举变量以 e 为前缀 
其他类型（如结构体）以 x 为前缀 
指针有一个额外的前缀 p , 例如short类型的指针前缀为 ps 
无符号类型的变量有一个额外的前缀 u , 例如无符号short类型的变量前缀为 us


#### 函数
文件内部函数以prv为前缀 
API函数以其返回值类型为前缀，按照前面对变量的定义 
函数的名字以其所在的文件名开头。如vTaskDelete函数在Task.c文件中定义 

#### 宏定义
宏名以所在的文件的文件名的一部分作为前缀（开头），并且用小写。 
比如, configUSE_PREEMPTION 在文件 FreeRTOSConifg.h 中. 
除了前缀，其余部分用大写，下划线来分隔单词。

#### 数据类型

基本数据类型可以直接使用，但是有如下的例外和规则：

char类型在每个平台都有其自身的定义方式。有些平台 char 等价于 signed char ，另一些则等价于 unsigned char，为此，要在代码中明确的使用 signed char 或 unsigned char 。直接使用 char类型是被禁止的。
不能直接使用 int 类型，要使用 short 和 long。
 float 和 double 没有在内核中使用，但是Demo 代码中有使用。

此外，有两种额外的类型要为每种平台定义。分别是：

portTickType

如果 configUSE_16_BIT_TICKS 被定义, 则 portTickType 被定义为无符号16bit 类型，否则为无符号 32 bit 类型。参考API文档中的 定制部分获取详细信息。


portBASE_TYPE
被定义为当前平台最佳的整形类型。例如，在一个 32 位的平台上， portBASE_TYPE 被定义为32 位的数据类型。在16位的平台上， portBASE_TYPE 则被定义为 16 位的数据类型。如果 portBASE_TYPE 被定义为 char 类型，则 必须为 signed char  类型，因为代码中用到这种类型作为一些函数的返回值类型，而返回值必须可以为负值以用来指示错误条件.

#### 编程风格
缩进
缩进使用 Tab . 一个 tab 等于 4 个空格.

#### 注释 
注释文字尽量不能超过 80 列，除非是用来描述一个参数。 
不采用 C++ 类型的注释（//）。 

---
# 二、源代码分析
## 1、任务的实现思路
a、任务的构成

一个任务，有任务名称、任务函数（实际要被执行的那部分代码，这个是我们自己写的）、优先级、堆栈等，这些都是在创建任务需要用到的。在FreeRTOS中，任务相关的这些信息保存在"tskTCB->pxStack"指向的堆栈中。再重复一下：“任务相关的这些信息保存在tskTCB->pxStack指向的堆栈中”。

tskTCB是一个结构体，下面是tskTCB（任务控制块）的定义：
```c
typedef struct tskTaskControlBlock /* The old naming convention is used to prevent breaking kernel aware debuggers. */
{
	volatile StackType_t	*pxTopOfStack;	/*< Points to the location of the last item placed on the tasks stack.  THIS MUST BE THE FIRST MEMBER OF THE TCB STRUCT. */
	ListItem_t			xStateListItem;	/*< The list that the state list item of a task is reference from denotes the state of that task (Ready, Blocked, Suspended ). */
	ListItem_t			xEventListItem;		/*< Used to reference a task from an event list. */
	UBaseType_t			uxPriority;			/*< The priority of the task.  0 is the lowest priority. */
	StackType_t			*pxStack;			/*< Points to the start of the stack. */
	char				pcTaskName[ configMAX_TASK_NAME_LEN ];/*< Descriptive name given to the task when created.  Facilitates debugging only. */ /*lint !e971 Unqualified char types are allowed for strings and single characters only. */
    //任务控制块还有很多东西，这里只做示意所以删掉了。
} tskTCB;
```

b、软件维护几个链表。

这些链表保存着不同状态任务的任务控制块,这个说法实际上不太严谨，因为链表中并没有保存整个TCB结构体的数据，而是只保存了“&( ( pxTCB )->xStateListItem )”，详细代码在task.c line238。
每个被创建的任务根据状态被添加到不同的链表中，实现任务的执行或者切换。系统每一次异常中断后切换任务。

```c
// 就绪任务链表 每个优先级对应一个链表
PRIVILEGED_DATA static List_t pxReadyTasksLists[ configMAX_PRIORITIES ];
// 延时任务链表
PRIVILEGED_DATA static List_t xDelayedTaskList1;                        
PRIVILEGED_DATA static List_t xDelayedTaskList2;                        
PRIVILEGED_DATA static List_t * volatile pxDelayedTaskList;
PRIVILEGED_DATA static List_t * volatile pxOverflowDelayedTaskList;
// 就绪任务链表，当任务调度器被挂起时，状态变换为就绪的任务先保存在此， 
// 恢复后移到 pxReadyTasksLists 中
PRIVILEGED_DATA static List_t xPendingReadyList;                
// 任务删除后，等待空闲任务释放内存
#if( INCLUDE_vTaskDelete == 1 )
    PRIVILEGED_DATA static List_t xTasksWaitingTermination;
    PRIVILEGED_DATA static volatile UBaseType_t uxDeletedTasksWaitingCleanUp = 
        ( UBaseType_t ) 0U;
#endif
// 被挂起的任务链表
#if ( INCLUDE_vTaskSuspend == 1 )
    PRIVILEGED_DATA static List_t xSuspendedTaskList;                   
#endif
```

## 2、创建任务流程（只标示出主要操作）

```c
BaseType_t xTaskCreate(	TaskFunction_t pxTaskCode,      //该任务要执行的函数名
                        const char * const pcName,		/*任务名称，lint !e971 Unqualified char types are allowed for strings and single characters only. */
                        const configSTACK_DEPTH_TYPE usStackDepth,      //任务堆栈大小  
                        void * const pvParameters,
                        UBaseType_t uxPriority,
                        TaskHandle_t * const pxCreatedTask ) //任务句柄，通过该句柄可以引用创建的任务。
{
    //a.根据堆栈增长方向确认如何申请内存
    #if( portSTACK_GROWTH > 0 )
    #else

    //b.申请堆栈内存，下面已STM32的向下增长为例
    pxStack = pvPortMalloc( ( ( ( size_t ) usStackDepth ) * sizeof( StackType_t ) ) ); 
    
    //c.申请任务控制块内存
    pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) );   
    
    //d.初始化一个新任务
    prvInitialiseNewTask( pxTaskCode, pcName, ( uint32_t ) usStackDepth, pvParameters, uxPriority, pxCreatedTask, pxNewTCB, NULL );
	
    //e.添加到任务就绪表中（通过任务控制块）    
    prvAddNewTaskToReadyList( pxNewTCB );
}                            
```                        

上面的代码中“d.初始化一个新任务”非常重要，着重说明

```c
static void prvInitialiseNewTask( 	TaskFunction_t pxTaskCode,
									const char * const pcName,		/*lint !e971 Unqualified char types are allowed for strings and single characters only. */
									const uint32_t ulStackDepth,
									void * const pvParameters,
									UBaseType_t uxPriority,
									TaskHandle_t * const pxCreatedTask,
									TCB_t *pxNewTCB,
									const MemoryRegion_t * const xRegions )
{
    //1、初始化堆栈为0xa5
    //2、获取栈顶保存在pxTopOfSatck
    //3、保存任务名字到pxNewTCB->pcTsakName
    //4、保存任务优先级pxNewTCB->uxPriority
    //5、初始化两个链表xStateListItem和xEventListItem
    //6、初始化任务控制块的一些变量
    //7、调用pxPortInitialiseStack()函数，初始化堆栈
    //8、保存任务句柄（任务控制块）
}                                    
```