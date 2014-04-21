# C++编程调试秘笈总结
---------
## 前言
在编写C++代码时，我们不应该自己捕捉缺陷，而是由编译器和可执行代码为我们做这些事情，该书便提供了这样的一个思考。作者以“调试器友好”的方式编写了一些方便安全检查时所需的宏代码并针对C++代码中最为常见的各种错误制定了一些规则，并用代码实现，使之很容易在运行时捕捉，或者尽可能地在编译时就捕捉缺陷。

## Chapter 1 C++的缺陷来自哪里
 C语言中存在的一些问题(目的是为了追求速度，产生高效的编译代码，并未考虑一些方便用户的特性，比如垃圾回收，越界检查等):
> * 程序员可以创建一定长度的数组，并可用一个超出数组边界的索引值访问元素
> * 滥用最多的是指针运算，程序员可以把指针运算所产生的任何值作为内存地址进行访问，不管该内存是否有效还是能否被访问
> * 程序员还可以在运行时使用`calloc()`和`malloc()`函数动态分配内存并使用`free()`函数负责释放内存。但是如果忘了销毁或者不小心销毁了多次，产生了内存泄露（分配内存后并未被释放，最终消耗完系统空间）或内存悬挂（释放对象后没有将指针置为NULL而之后又解引用了它，未定义的指针解引用是非常严重的）等灾难性的问题

C++语言中存在的一些问题:
> * 友元和多重继承并不是个很好的思路
> * 混用了`new`和`delete`，其中一个带方括号和一个不带方括号，正确的形式如下：
```
A* p_object=new A();
A* p_array=new A[size];
delete p_object;
delete []p_array;
```

## Chapter 2 什么时候捕捉缺陷
在编译时诊断错误，有如下规则:
> * 禁止隐式类型转换：关键字explicit声明一个接受一个参数的构造函数，并禁止使用转换操作符 
> * 用不同的类表示不同的数据类型
> * 不要使用单纯功能的枚举创建整形常量，而是用它们创建新类型

为什么呢，下面一一解释
1 假设我们有两个类A和B，并有一个期望接受一个B类型的参数的函数：
```C++
void doSomething(const B& b)
```
但是我们不小心向它提供了A类型的对象：
```C++
A a(input);
doSomething(a);
```
 某些情况，这样的代码可通过编译，原因是它有可能平静的进行隐式类型转换：A转换成B。可能通过以下两种方式发生

 1. B类接受含A类型的参数构造函数,它可以隐式地把A转换为B
```C++ 
class B {
    public:
        B(const A& a);
}
```
 2. A类具有一个可以将其转换为B的操作符，以明确的方式提供了转换方法
```C++
class A{
    public:
//转换操作符operator type()：type可以是基本数据类型，类，结构体
         operator B() const; 
}
```
所以针对上述问题，对于所有接受一个参数的构造函数用关键字`explicit`声明，也不建议用转换操作符，这是值得推荐的做法。一般而言，隐式转换的所有可能性都是不好的思路，还记得深入计算机系统第二章讲过FreeBSD开源系统曾出现的getpeername的安全漏洞么，这是由于无符号数和有符号数间的不匹配造成了隐式类型转换。不过我们还可以用另外一个方法进行转换
```C++
class A{
    public:
         B asB() const; 
}

A a(input);
doSomething(a.asB()); //  显式转换
```

2 我们定义了两个枚举，分别表示一周中的某天及月份,这些常量都是整数。假设我们有一个期望接受一周中的某天作为参数的函数
```C++
enum {SUN1,MON=1,TUE,WED,THU,FRI,SAT};
enum {JAN=1,FEB,...,DEC};

void func(int day_of_week);
```
因而下面调用将不会产生任何警告的情况下通过编译：```func(JAN);```
所以捕捉此类缺陷的办法就是创建新类型的枚举，直接限定了新类型的枚举范围，这样就可以在编译时判断是否有错误。
```C++
typedef enum {SUN1,MON=1,TUE,WED,THU,FRI,SAT} DayofWeek;
typedef enum {JAN=1,FEB,...,DEC} Month;
```

## Chapter 3  在运行时遇见错误该怎么办
我们把精力集中在运行时的一类错误--缺陷。为了捕捉缺陷专门编写的一段代码称为安全检查，当其失败时，就表示发现了缺陷，哪如何处理呢，这里作者提供这样的一个思路

 1. 定义一个SCPP_ASSERT宏，永久性的安全检查，用来捕捉运行时错误，并提供与错误有关的具体信息
```C++
#scpp_assert.h
#define SCPP_ASSERT(condition,msg) \
    if(!(condition)) {             \
	std:ostringstream s;       \
	s << msg;                  \
SCPP_AssertErrorHandler(_FILE_,_LINE_,s.str()_c_str()); \
}

#scpp_assert.cpp
void SCPP_AssertErrorHandler(const char *file_name,
                             unsigned line_no,
                             const char *msg){
//此处适合插入断点，合适情况下还可向一个日志文件写入相同的信息
#ifdef SCPP_THROW_EXCEPTION_ON_BUG
     throw scpp::ScppAssertFailedException(file_name,
                                           line_no,msg);
#else
    cerr << msg << "in file "<<file_name << 
                   " #" <<line_no <<endl<<flush;
    exit(1);
#endif
}

#scpp_assert.h
#ifdef SCPP_THROW_EXCEPTION_ON_BUG
#include<exception>

namespace scpp {
	class ScppAssertFailedException :public std::exception {
		private:
			std::string what_;
		public:
			ScppAssertFailedException(const char *file_name,                                      
			                          unsigned line_no,   
                                                  const char *msg);
			virtual void const char* getwhat() const throw()          { return what_.c_str();}
			virtual ~ScppAssertFailedException() throw() {}
	}

}

#scpp_assert.cpp
#ifdef SCPP_THROW_EXCEPTION_ON_BUG
namespace scpp {
	ScppAssertFailedException::ScppAssertFailedException(const char *file_name,
			                                     unsigned line_no,
			                                     const char *msg) {
		ostringstream s;
		s << "SCPP Assertion failed with message " <<msg <<" in file " <<file_name << " # "<<line_no;
		what_=s.str();
	}
}
#endif
```
我们可以看到该宏接受一个条件和一条错误信息。条件为真不执行任何事情，为假时，错误信息会输出到`ostringstream`中，并且错误处理函数将被调用。这里有两个问题：

*  问：为什么要调用scpp_assert.cpp文件中一个单独AssertErrorHandler函数，而不是在scpp_assert.h文件的宏中执行相同的操作
   答：调试器更擅长对函数而不是宏进行逐步调试
*  问：为什么AssertErrorHandler函数向我们提供了两种选择机会，要么终止程序，要么抛出一个异常
   答：在最常见的情况下我们发现第一个缺陷时默认采取的办法是终止程序，修补缺陷并再次开始，这时候将打印出错误信息并终止程序，即对应没有定义的SCPP_THROW_EXCEPTION_ON_BUG符号。那么定义了该符号的情况呢，在某些情况下，有部分安全检查必须保留在代码中，即使是在产品模式下。假设有一个持续依次处理大量请求的程序在处理某个请求时安全检查失败，终止程序并不是理想的选择，应该采取的办法是抛出一个异常，包含详细的错误信息并把错误信息记录在某日志文件中，可能还需要发送邮件或警报，宣布对当前请求的处理失败，同时继续处理发送其他的请求。







