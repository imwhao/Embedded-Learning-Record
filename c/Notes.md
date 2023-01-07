>对C语言知识进行复习，为嵌入式学习做准备。
>视频教程选用[C语言程序设计_浙江大学_中国大学MOOC(慕课) ](https://www.icourse163.org/course/ZJU-9001)


# C语言的发展与版本

- K&R the C （经典C）
- 1989年提出ANSI C
- 1990年ISO接受ANSI标准，即C89
- C99 （支持任意位置定义变量，支持const等）
- C11

# C语言基本知识
[教程](https://www.icourse163.org/course/ZJU-9001)前八周内容（判断、循环、数据类型、函数、数组等）由于大学学习阶段已经掌握，所以采用看PDF课件的方法较快复习。第九周以后的内容（指针、结构体、文件、链表等）由于较为生疏，所以看视频教程学习。

基本知识学习中遇到的一些细节：
- void f(void) 还是void f()?
	- void f()在传统C中表示f函数的参数表未知，并不表示没有参数
	- 推荐 void f(void)
- C语言不允许函数的嵌套定义
- int i, j, sum(int a, int b);
	- 混合使用变量定义和函数声明。（不建议这样写）
- 数组变量本⾝身不能被赋值。要把⼀一个数组的所有元素交给另⼀一个数组，必须采⽤遍历。
- 二维数组初始化列数必须给出，行数可以省略。如：`int a[][5]={{0,1,2,3,4},{2,3,4,5,6},};`

# 指针
指针的定义：`int* p;`

`&x`含义为取x的地址。`*p`的含义为取指针p的值。（注意：在定义指针p后在它未指向变量时不能使用`*p`赋值，因为此时p指向的是未知随机地址）

指针与const：指针和所指都可以为常量const。
1. 当指针为const时：`int* const q = &i;`，变量不能再指向其他变量。
2. 当变量为const时：`const int *p = &i;`，表示不能通过这个指针去修改变量i（不表示变量i为const）。
	1. 应用：当后面自定义数据结构传参时，可以用指针传递，这样能用较少的字节实现参数传递，变量为const的使用保证不会再函数内部对其进行修改。
	2. `const int a[] = {1,2,3,4,5,6};`数组a已经为const指针，前面又用const限定变量，所以这样的数组必须通过初始化进行赋值。参数传递时使用const数组可以防止函数内对数组进行修改。

`int*p`，p+1的含义是加上`sizeof(*p)`，指针相减的结果是地址之差除以`sizeof(*p)`。

`*p++`的含义是去除p所指的数据来，之后把p移到下一个地址。

动态内存分配`void* malloc(size_t size);` 使用：
```c
#include <stdlib.h>
int *a;
a = (int*)malloc(number*sizeof(int)) //申请空间
free(a); //释放空间
```
内存释放`free()`函数使用注意只能释放**申请**的**首地址**空间。

指针应用场景：
1. swap()函数交换两个变量的值。函数调用传递的是值。在使用指针前，函数内部无法改变外部的变量。通过指针可以使函数内部对外部的变量进行操作。
2. 函数调用时数组参数传递的实际是指针，即数组地址。可以通过指针对数组进行修改。
	1. **实际上，数组变量是特殊的常量指针**，所以数组之间不能直接赋值。
3. 需要返回多个值。传入的指针参数实际是需要保存返回结果的变量。
4. 当常见的值都有可能为正常值时，return难以返回函数状态，指针作为函数状态的返回。
5. 动态内存申请。

# 字符串
C语言的字符串是以字符数组形式存在的，末尾以`'\0'`作为结束标志。(由于结束标志的存在，字符数组最大能存储字符数为数组长度-1)
- 不能用运算符对字符串进行运算。
- 通过数组方式可以遍历字符串。

如果要构造一个字符串，使用数组。（`char *p = "hello, world!";`）
如果要处理一个字符串，使用指针。（`char p[] = "hello, world!";`）

注：`char* s = "Hello, world!"`，s作为指针实际上是`const char* s'`，历史原因编译器接受不带const的写法。但是试图对s所指的字符串进行写入会导致严重的后果。如果需要修改字符串，应该用数组`char [] = "Hello, world!"`。 

字符串可以表示为`char*`的形式，但`char*'`形式只有结尾有结束符`'\0'`时才表示它是字符串。

字符串函数：
1. `size_t strlen()` 返回字符串长度，不包括结尾0（`sizeof()`包括结尾0）。
2. `int strcmp(const char *s1, const char *s2)` 比较两个字符串，返回
	1. 0:s1\==s2
	2. 1:s1>s2
	3. -1:s1\<s2
	4. 其他版本：`int strncmp(const char *s1, const char *s2, size_t n)`指定判断前n个字符。
3. `char* strcpy(char* restrict dst, const char* restrict src);`
	1. 把src的字符串拷贝到dst
	2. restrict表示src和dst不重叠（C99）
	3. 返回dst（为了能作为参数链起代码来）
	4. 复制一个字符串代码：`char *dst = (char*)malloc(strlen(src)+1);strcpy(dst,src);` 申请内存时+1是因为strlen所得长度不包含结束符
	5. 安全版本`char* strncpy(char* restrict dst, const char* restrict src, size_t n);`指定最多拷贝字符数目。
4. `char* strcat(char *restrict s1, const char *restrict s2);`
	1. 把s2拷贝到s1的后面，接成一个长的字符串。
	2. 返回s1
	3. s1必须有足够的空间
	4. 安全版本`char* strncat(char *restrict s1, const char *restrict s2, size_t n);`指定最多拷贝字符数目。
5. 字符串搜索函数
	1. `char * strchr(const char *s, int c);`在字符串中找字符，返回找到的第一个字符的地址。（从左向右找）
	2.  `char * strrchr(const char *s, int c);`在字符串中找字符，返回找到的第一个字符的地址。（从右向左找）
	3. `char * strstr(const char *s1, const char *s2);`在字符串中找字符串。
	4. `char * strcasestr(const char *s1, const char *s2);`在字符串中找字符串。（忽略大小写）


# 枚举
枚举是一种用户定义的数据类型，以如下语法来声明：`enum 枚举类型名字 {名字0,...,名字n};`，如 `enum colors {red, yellow, green};`。枚举类型名字通常并不真的使用，要用的是在大括号里的名字，因为它们就是常量符号，类型是int，值依次从0到n。

枚举实现的是用符号来表示数字。对于有意义排比的名字，用枚举比 `const int` 方便。

小技巧——自动计数的枚举：`enum COLOR {RED, YELLOW, GREEN, NumCOLORS};` NumCOLORS变量的值即为枚举的总数。

# 结构体（struct）
## 结构体的三种声明方法
```c
// 1. p1和p2都是point，里面有x和y的值
struct point
{
    int x;
    int y;
};
// 2. p1和p2都是一种无名结构，里面有x和y
struct
{
    int x;
    int y;
} p1, p2;
// 3. p1和p2都是point，里面有x和y的值
struct point
{
    int x;
    int y;
} p1, p2;
```

## 结构体的赋值方法
```c
struct date
{
    int month;
    int day;
    int year;
};
struct date today ={07,31,2014};
struct date thismonth = {.month=7, .year=2014};
```

## 结构体与数组区别
1. 结构体的名字和地址指针不同，数组的名字就是地址指针。
2. 结构体和数组的访问方式不同。
3. 结构体变量之间可以相互赋值，数组变量之间不能相互赋值。
4. 结构体的成员类型可以不相同，数组的每个元素类型必须相同。
5. 结构体定义的时候不分配空间，数组定义时即分配空间。
6. **结构体作为参数传递进函数后是在函数内新建一个结构变量，并复制调用的结构的值**，数组传递的是地址。

## 结构体函数传参的两种方法
1. 直接将结构体的名字传入函数，在函数内部为临时变量，函数结束后不改变原结构体的值，可以用return 结构体的方式对原结构体进行改变。
2. 将结构体指针作为参数传入函数中是处理结构体更有效的一种方法，可以节省时间和空间。

## 访问结构体指针所指结构体成员的两种方法
```c
struct date {
int month; int day; int year;
} myday;
struct date *p = &myday;
(*p).month =12; // 方法1
p->month = 12; //方法2
```

## 类型定义
可以使用 `typedef`来进行变量名重载（即为现有数据类型创建一个新名字）从而简化结构体声明。

```c
typedef long int64_t // 用int64_t来表示long
typedef struct ADate{
	int month; int day; int year;
}Date;
// 上面结构体声明中ADate可以省略
int64_t i = 10000000;
Date d = {9, 1, 2005};
```

# 联合/共用体（Union）
定义方式：`union 共用体名{  成员列表  };`

结构体和共用体的区别在于：结构体的各个成员会占用不同的内存，互相之间没有影响；而共用体的所有成员占用同一段内存，修改一个成员会影响其余所有成员。  

结构体占用的内存大于等于所有成员占用的内存的总和（成员之间可能会存在缝隙），共用体占用的内存等于最长的成员占用的内存。共用体使用了内存覆盖技术，同一时刻只能保存一个成员的值，如果对新的成员赋值，就会把原来成员的值覆盖掉。

共用体在一般的编程中应用较少，在单片机中应用较多。

# 位操作
左移`<<`，右移`>>`。 负数右移补1，其余都补0.

```c
uint32_t temp = 0;
//给位6赋值为1的两种方法：
temp &= 0xFFFFFFBF; temp |= 0x00000040; // 法1
temp &= ~(1<<6); temp |= 1<<6; // 法2
// 按位异或控制位6翻转
temp ^= 1<<6;
```

# 宏定义
`#define 标识符 字符串`
标识符：宏定义的名字。
字符串：常数、表达式、格式串等。

宏定义的核心是替换。

带参数的宏定义：
```c
#define LED1(x) do{x ?\
	HAL_GPIO_WritePin(LED1_GPIO_PORT, LED1_GPIO_PIN, GPIO_PIN_SET):\
	HAL_GPIO_WritePin(LED1_GPIO_PORT, LED1_GPIO_PIN, GPIO_PIN_RESET);}while(0)
```
带参数的宏定义使用大括号while(0)可以防止受到其他代码大括号、分号、运算符优先级的影响，保证按你期望的方式调用。

# 条件编译
让编译器只对满足条件的代码进行编译，不满足条件的不参与编译。
- \#if 编译预处理指令，类似if
- \#ifdef 判断某个宏是否已被定义
- \#ifndef 判断某个宏是否未被定义
- \#elif 若前面的条件不满足，则判断新的条件，类似else if
- \#else 若前面的条件不满足，则执行后面的语句，类似else
- \#endif \#if, \#ifdef, \#ifndef的结束标志


# extern声明
放在函数/变量前，表示此函数/变量在其他文件定义，以便本文件引用。
```c
extern uint16_t g_usart_rx_sta;
extern void delay_us(uint32_t nus); //也可以用在函数声明
```