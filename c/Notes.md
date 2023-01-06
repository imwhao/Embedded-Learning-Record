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