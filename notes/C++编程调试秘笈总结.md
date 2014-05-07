# C++编程调试秘笈总结
---------
## 前言
在编写C++代码时，我们不应该自己捕捉缺陷，而是由编译器和可执行代码为我们做这些事情，该书便提供了这样的一个思考。作者以“调试器友好”的方式编写了一些方便安全检查时所需的宏代码并针对C++代码中最为常见的各种错误制定了一些规则，并用代码实现，使之很容易在运行时捕捉，或者尽可能地在编译时就捕捉缺陷。

## Chapter 1 C++的缺陷来自哪里
 C语言中存在的一些问题(目的是为了追求速度，产生高效的编译代码，并未考虑一些方便用户的特性，比如垃圾回收，越界检查等):
> * 程序员可以创建一定长度的数组，并可用一个超出数组边界的索引值访问元素
> * 滥用最多的是指针运算，程序员可以把指针运算所产生的任何值作为内存地址进行访问，不管该内存是否有效还是能否被访问，如解引用NULL指针`strlen(NULL)`将会导致程序崩溃
> * 程序员还可以在运行时使用`calloc()`和`malloc()`函数动态分配内存并使用`free()`函数负责释放内存。但是如果忘了销毁或者不小心销毁了多次，产生了内存泄露（分配内存后并未被释放，最终消耗完系统空间）或内存悬挂（释放对象后没有将指针置为NULL而之后又解引用了它，未定义的指针解引用是非常严重的）等灾难性的问题
> * `sprintf()`和某些字符串函数在写入缓冲区时，它们可能会改写越过缓冲区尾部的内存，从而导致不可预料的程序行为；对于对应的安全版本会安静地在缓冲区结束时截断，但很可能不是我们所期望的结果，因而多用C++的`string`和`stringstream`（**其实关于C的字符串操作函数和C++ 的`string`、`stringstream`孰优孰劣还是有争论的，有空的话不妨研究一番**）

C++语言中存在的一些问题:
> * 友元和多重继承并不是个很好的思路
> * 混用了`new`和`delete`，其中一个带方括号和一个不带方括号，正确的形式如下：
```
A* p_object=new A();
A* p_array=new A[size];
delete p_object;
delete []p_array;
```
**个人思考：语言的设计源于实际的需求，设计什么样的特性是有舍有得的，设计思想和特性决定了它能做什么事，不能做什么事，有怎么的好处也有相应的缺陷，任何语言都不是silver bullet ，你不能单纯说它好坏. 只有当认识清楚语言背后的设计思想、演化史，了解各自的特性和缺点，就不会出现 遇到具体问题而直接掉入编程语言的坑了**

**深入阅读：
Unix哲学编程艺术（Unix的设计思想是很值得思考和借鉴的）
C++语言的设计和演化、Java语言的演化设计史（虚拟机、设计模式，对比Java和C++的不同点）
还有Python语言它有着怎样不同的设计等，最后再学习下函数式编程语言它是怎么思考的（表示一直不理解）**

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
    public：
//转换操作符operator type()：type可以是基本数据类型，类，结构体
         operator B() const; 
}
```
所以针对上述问题，对于所有接受一个参数的构造函数用关键字`explicit`声明，也不建议用转换操作符，这是值得推荐的做法。一般而言，隐式转换的所有可能性都是不好的思路，还记得深入计算机系统第二章讲过FreeBSD开源系统曾出现的getpeername的安全漏洞么，这是由于无符号数和有符号数间的不匹配造成了隐式类型转换。不过我们还可以用另外一个方法进行转换
```C++
class A{
    public：
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
我们把精力集中在运行时的一类错误--缺陷。为了捕捉缺陷专门编写的一段代码称为安全检查，当其失败时，就表示发现了缺陷，那如何处理呢，这里作者提供这样的一个思路

 1. 定义一个SCPP_ASSERT宏，永久性的安全检查，用来捕捉运行时错误，并提供与错误有关的具体信息
```C++
#scpp_assert.h
#define SCPP_ASSERT(condition,msg) \
    if(!(condition)) {             \
	std:ostringstream s;       \
	s << msg;                  \
SCPP_AssertErrorHandler(__FILE__,__LINE__,s.str().c_str()); \
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

#scpp.h
#ifdef SCPP_THROW_EXCEPTION_ON_BUG
#include<exception>

namespace scpp {
	class ScppAssertFailedException :public std::exception {
		private:
			std::string what_;
		public:
			ScppAssertFailedException(const char *file_name,                                      unsigned line_no,   
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
   答：在最常见的情况下我们发现第一个缺陷时默认采取的办法是终止程序，修补缺陷并再次开始，这时候将打印出错误信息并终止程序，即对应没有定义的SCPP_THROW_EXCEPTION_ON_BUG符号。那么定义了该符号的情况呢，在某些情况下，有部分安全检查必须保留在代码中，即使是在产品模式下。假设有一个持续依次处理大量请求的程序在处理某个请求时安全检查失败，终止程序并不是理想的选择，应该采取的办法是抛出一个异常，包含详细的错误信息并把错误信息记录在某日志文件中，可能还需要发送邮件或警报，宣布对当前请求的处理失败，同时继续处理发送其他的请求。因而在scpp_assert.h声明了一个异常类

 * 问：什么时候编写安全检查？
  答：如果我们的想法是等我们编码好后再回过头来添加安全检查，这个计划可能永远不糊实施。较好的建议是从一开始编写新函数新类新功能时等具体的代码前就应该为它所有的输入编写好安全检查和测试。可以看出编写安全检查并不困难，它不仅让你更明确你所要做的工作，更重要的是它会在以后的测试阶段得到足够的回报，这要比你以后回过头来调试代码要方便得多。
**注：要养成这样的习惯，单元测试也是类似的思路：编码同时编写好安全检查和测试,更明确的办法是当我们开始编写具体的代码前为它的所有输入编写安全检查**
如下类似的代码用来测试：
```
#include <iostream>
#include "scpp_assert.h"

using namespace std;
int main(int argc,char *argv[]) {
	cout << "Hello,SCPP_ASSERT" <<endl;

	try {
		double price=100.0 ; //合理价格
		SCPP_ASSERT(0< price && price <=1e6,"Stock price " <<price <<" is out of range "); //条件成立时不执行

		price=-1;
		SCPP_ASSERT(0< price && price <=1e6,"Stock price " <<price <<" is out of range "); //条件不成立时执行并捕获异常
    } catch (const exception& ex) {
    	cerr << "Exception caught in " << _FILE_ << " # "<< _LINE_ << ". "<< endl;
    	cerr << ex.what() <<endl;
    }
    return 0;
}

//在SCPP_ASSERT宏中也可使用任何类的对象，只要它定义了<<操作符，设计和测试如下：
/* Test :
*MyClass obj(inputs);
*SCPP_ASSERT(obj.IsValid(),"Object "<< obj <<" is invalid.");
*/
class MyClass {
public:
	bool IsValid() const ; //对象状态有效即返回true
	//Implement constructors 、destructors 
private:
	int data;
	friend std::ostream operator <<(std::ostream& os ,const MyClass& obj);
}
inline std::ostream operator <<(std::ostream& os ,const MyClass& obj) {
	//执行一些任务，按被人理解的格式显示对象
	os << obj.data;
	return os;
}
/*
* Output : 
* Hello,SCPP_ASSERT
* Exception caught in xxx.cpp #13 .
* SCPP assertion failed with message 'Stock price -1 is out of range ' in file xxx.cpp #13
*/
```
 * 问：什么时候使用它
  答：我们意识到代码中可能含有大量的安全检查，有些是永久性的，有些是临时性的。为了保持C++代码执行的高效性和有效性，在不同运行阶段执行不同的策略：
1 在Debug模式，打开测试安全检查，对错误进行调试
2 在Release模式，打开测试安全检查，快速调试（考虑到1的安全检查会较慢）
3 在Release模式下关闭安全检查，发布产品
代码实现如下：
```C++
#scpp_assert.h
#ifdef _DEBUG
#define SCPP_TEST_ASSERT_ON
#endif

#ifdef SCPP_TEST_ASSERT_ON
#define SCPP_TEST_ASSERT(condition,msg) SCPP_ASSERT(condition,msg)
#else
#define SCPP_TEST_ASSERT(condition,msg)
```
可以看到SCPP_ASSERT是永久性的安全检查，SCPP_TEST_ASSERT可以在编译时打开

下面分别就索引越界、编写一致的比较操作符、未初始化变量 指针操作、内存泄露等缺陷进行一一处理，用于在代码进入产品阶段前捕捉各类缺陷。

## Chapter 4 索引越界
避免"索引越界"错误的规则如下（针对C++）：
> * 不要使用静态或动态分配的数组，改用array或vector模板
> * 不要使用带**方括号**的new和delete操作符，让vector模板为多个元素分配内存
> * 使用scpp::vector代替std::vector,使用scpp::array代替静态数组，并打开安全检查（自动在使用下标访问提供了索引边界检查）

C++中创建类型T的对象的数组：
```C++
#define N 10
T static_arr[N]; //数组长度在编译时已知

int n=20;
T* dynamic_arr=new T[n]; //数组长度在运行时计算

std::vector<T> vector_arr; //数组长度在运行时进行修改
```

 **1.动态数组**
采用的办法是继承std::vector<T>,并重载<< []运算符，提供一个能够捕捉越界访问错误的实现，实现代码和测试如下：
```C++
//scpp_vector.h
#ifndef  _SCPP_VECTOR_
#define  _SCPP_VECTOR_

#include <vector>
#include "scpp_assert.h"

namespace scpp {

	//wrapper around std::vector,在[]提供了临时的安全检查：重载[] <<运算符
	template<typename T>
	class vector : public std::vector<T> {
		public:
		 	typedef unsigned size_type;

		 	//常用的构造函数 commonly use cons
		 	explicit vector(size_type n=0) : std::vector<T>(n) {

		 	}
		 	vector(size_type n,const T& value) : std::vector<T>(n,value) {

		 	}

		 	template <class InputIterator> vector(InputIterator first,InputIterator last) 
		 		: std::vector<T>(first,last) {

		 	}
		 	
		 	//Note : we don't provide a copy-cons and assignment operator  ?

			//使用scpp::vector提供更安全的下标访问实现，它可以捕捉越界访问错误
		 	T& operator[] (size_type index) {
		 		SCPP_ASSERT( index < std::vector<T>::size() ,
		 			"Index " << index << " must be less than " << std::vector<T>::size());
		 		return std::vector<T>::operator[](index);
		 	}

		 	//? difference 
		 	const T& operator[] (size_type index) const {
		 		SCPP_ASSERT( index < std::vector<T>::size() ,
		 			"Index " << index << " must be less than " << std::vector<T>::size());
		 		return std::vector<T>::operator[](index);
		 	}

		 	//允许此函数访问这个类的私有数据
		 	//friend std::ostream& operator<< (std::ostream& os,const ) ?
			};
} //namespace

template<typename T>
inline  std::ostream& operator<< (std::ostream& os,const scpp::vector<T>& v) {
	for(unsigned i=0 ;i<v.size();i++) {
			os << v[i];
			if( i+1 < v.size()) os << " ";
	}
	return os;
}


#endif

//test_vector.cpp
#include "scpp_vector.h"
#include <iostream>

using namespace std;
int main() {
	//usage-创建一个具有指定数量的vector：scpp::vector<int> v(n); 把n个vector元素都初始化为一个值：scpp::vector<int> v(n，val)
	//方法3：scpp::vector<int> v; v.reserve(n),表示开始的vector是空的，对应的size()为0,
    //并且开始添加元素时，在长度达到n之前，不会出现导致速度降低的容量增长现象
	scpp::vector<int> vec;
    for(int i=0;i<3;i++){
    	vec.push_back(4*i);
	}
	cout << "The vector is : "<< vec <<endl;

	for(int i=0;i<=vec.size();i++) {
		cout << "Value of vector at index " << i << " is " << vec[i] <<endl;
	}
	return 0;
}
```
我们直接使用scpp::vector而尽量不与std::vector交叉使用
 **2.静态数组**
静态数组是在堆栈上分配内存，而vector模板是在构造函数中用new操作符分配内存的，速度相对慢些，为保证运行时效率，建议使用array模板（同样也是堆栈内存），实现代码和测试如下：
```C++
//scpp_array.h
#ifndef _SCPP_ARRAY_H_  
#define _SCPP_ARRAY_H_

#include "scpp_assert.h"

namespace scpp {

//wrapper around std::vector,在[]提供了临时的安全检查
//fixed-size array
template<typename T,unsigned int N>
class array {
	public:
		 typedef unsigned int size_type;

		 //常用的构造函数 commonly use cons
		array() {}
		explicit array(const T& val) {
		 		// for(size_type i=0;i<size();i++) {
		 		// 	data[i]=val;
		 		// }
			for(unsigned int i=0;i< N;i++) {
		 			m_data[i]=val;
		 		}
		}
		 		
		size_type size() const { 
			return N;
		} //must use const if we use the size()
		 	
		//Note : we don't provide a copy-cons and assignment operator  ?

		T& operator[] (size_type index) {
		 	SCPP_ASSERT( index < N,
		 			"Index " << index << " must be less than " << N);
		 	return m_data[index];
		 }

		 //? difference 
		const T& operator[] (size_type index) const {
		 	SCPP_ASSERT( index < N ,
		 			"Index " << index << " must be less than " << N);
		 	return m_data[index];
		}

		 //模拟迭代器的begin和end方法
		 //访问方法accessors
		T* begin() { 
			return &m_data[0];
		}

		const T* begin() const { 
			return &m_data[0];
		}

		 //返回越过数组尾部的迭代器
		T* end() { 
		 	return &m_data[N];
		}

		const T* end() const { 
		 	return &m_data[N];
		}
	private:
		T m_data[N];
	};
} //namespace scpp

template<typename T,unsigned int N>
inline  std::ostream& operator<< (std::ostream& os,const scpp::array<T,N>& v) {
	for(unsigned int i=0 ;i< N;i++) {
			os << v[i];
			if( i+1 < v.size()) os << " ";
	}
	return os;
}
#endif

//test_array.cpp
#include "scpp_array.h"
#include <iostream>
#include <algorithm> //sort algorithm
using namespace std;
int main() {
	//use vector/array class instead of static array or dynamic array
	scpp::array<int,5u > arr(0); 
	arr[0]=7;
	arr[1]=2;
	arr[2]=3;
	arr[3]=9;
	arr[4]=0;

	cout<< "Array before sort : "<< arr <<endl;
	sort(arr.begin(),arr.end());
	cout<< "Array after sort : "<< arr <<endl;

    arr[5]=8;
	return 0;
}
```
 **3.多维数组**
 略

## Chapter 5 未初始化的变量-编写一致的比较操作符
**1. 未初始化的数值**
避免未初始化的变量，尤其是类的数据成员，有如下规则：
> * 不要使用像int、unsigned、bool等内置类型作为类的数据成员，反之要模仿std::string类默认构造函数可初始化string为空字符串的行为，使用Int、Unsigned、Bool等类，因为这样可以避免在构造函数对它们进行初始化
> * 用这些新类代替内置类型表示传递给函数的参数，就自动获得了编译时类型安全的优点

假设有一个MyClass类，我们决定添加一个新的数据成员int_date,我们已经在构造函数初始化了很多数据成员，如果我们忘记了对新数据成员初始化，它可能填充了垃圾信息，对于这样的错误，我们该怎么办呢,即采用上面的规则：

* 不要使用int，用Int
* 不要使用double,用Double
* 不要使用float，用Float

设计和测试的代码如下：
```
#ifdef _SCPP_TYPE_
#define _SCPP_TYPE_

#include "scpp_assert.h"

//封装内置类型的模板类，初始化默认为0
tmplate<typename T>
class TNumber {
private:
	T data;
public:
	TNumber(const T& x=0):  data(x) {
    }
    operator T() const {return data;}
    //TNumber(const TNumber& type):data(type.data){} 没有必要实现

    TNumber& operator =(const T& x){
    	data=x;
    	return *this;
    }

    //后缀操作符
    TNumber operator ++(int) {
    	TNumber<T> copy(*this);
    	++data;
    	return copy;

    }

    //前缀操作符
    TNumber operator ++(){
    	++data;
    	return *this;
    }

    T operator /(T x) {
    	SCPP_ASSERT(x!=0,"Attempt to divide by 0");
    	return data/x;
    }

    TNumber operator +=(T x) {
    	data +=x;
    	return *this;
    }

    TNumber operator -=(T x) {
    	data -=x;
    	return *this;
    }

    TNumber operator *=(T x) {
    	data *=x;
    	return *this;
    }
 
    TNumber operator /=(T x) {
    	SCPP_ASSERT(x!=0,"Attempt to divide by 0");
    	data /=x;
    	return *this;
    }
    
  friend std::ostream& operator <<(std::ostream& os,const TNumber& type);
};
inline std::ostream& operator <<(std::ostream& os,const TNumber& type) {
	os << type.data;
	return os;
}
typedef long long int64;
typedef unsigned long long unsigned64;

typedef TNumber<int> Int;
typedef TNumber<unsigned> Unsigned;
typedef TNumber<int64> Int64;
typedef TNumber<unsigned64> Unsigned64;
typedef TNumber<float> Float;
typedef TNumber<double> Double;
typedef TNumber<char> Char;
#endif

//test.cpp

class MyClass{
private:
	Int data_int;
	std::string text;
public:
	MyClass(){ }
	explicit MyClass(const A& a) {}
	MyClass(const string& text,double weight) :text(text) {//...}
}
```
首先，接受任何内置类型T的构造函数，没有声明为explicit是有意为之的，它自动初始化为0
其次，operator T()转换操作符允许把这个类的实例隐式转换为对应的内置类型，这也是有意设计的，可以方便来回转换
再者,使用内置类型，试图除零可能出现错误，因而SCPP_ASSERT错误处理函数将被调用
最后总结一句：**健壮的代码不应该在变量初始化前使用，因而让未初始化的变量具有零的安全值显然要比产生的随机垃圾值好得多**

**2. 未初始化的布尔值**
```C++
class Bool{
private:
    bool state;

public:
    Bool(bool x=false) ：state(x) {}
    operator bool() const { return state;}

    Bool& operator =(bool x) {
        state=x;
        return *this;
    }

    Bool& operator &=(bool x) {
        state &=x;
        return *this;
    }

    Bool& operator |=(bool x) {
        state |=x;
        return *this;
    }
    friend std::ostream& operator <<(std::ostream& os, Bool b);
};

inline std::ostream& operator <<(std::ostream& os,const TNumber& type) {
	if(b)
		os << "True";
	else 
		os << "False";
	return os;
}
```
它被初始化为false,并可打印出True和False,而不是0、1,这样显得更为清晰，更容易理解

综合1和2，使用这些类不仅可以避免在多个构造函数对其初始化，还具有编译时类型安全的特点。
假设有一个unsigned类型参数的函数：```void func(unsigned u)```
下面代码就可以通过编译:```int i=0; func(i);```,这样类型就不安全。如用我们定义的类，情况则不会如此,编译不通过：
```C++
void func(Unsigned u)
Int i=0; func(i);
```

**3.编写一致的比较符**
该问题源于我们编写了一个新类MyClass,有时候需要编写如下的表达式
```
MyClass x,y;
if(x<y) {//Do A;}
else if(x==y) {//Do B}
else {//Do C}
```
有时候我们并不需要比较操作符，但不小心在STL操作中使用了我们的类，例如我们试图对类的实例进行排序，就会犯这样的错误，忘记定义这些比较操作符
```
vector<MyClass> v;
v.push_back(MyClass(7));
v.push_back(MyClass(1));
v.push_back(MyClass(4));
sort(v.begin(),v.end());
```
试图编译这段代码随即产生各种错误，不难理解就是要定义<运算符。所以作者编写了一个通用的宏，规则如下：

* 编写CompareTo函数，并用SCPP_DEFINE_COMP_OPEARTORS宏实现所有的比较操作符
```C++
#scpp_types.h
//该宏以一致的方式定义了所以6个比较操作符,因而只需要在类中修改CompareTo函数
#define SCPP_DEFINE_COMP_OPERATORS(Class) \
bool operator <(const Class& that) const {return CompareTo(that)< 0;}  \
bool operator >(const Class& that) const {return CompareTo(that)> 0;}  \
bool operator ==(const Class& that) const {return CompareTo(that)==0;}  \
bool operator <=(const Class& that) const {return CompareTo(that)<=0;}  \
bool operator >=(const Class& that) const {return CompareTo(that)>=0;}  \
bool operator !=(const Class& that) const {return CompareTo(that)!=0;}        

#xxx.cpp
class MyClass{
private:
	Int data_int;
public:
	MyClass(){ }
	
    int CompareTo(const MyClass& that) const {
		//Implement as you need
	}
}
```
## Chapter 6 无效的指针、引用和迭代器

说到指针，我们创建一个包含10个元素的vector,并出于某种原因决定保存一个指向索引位置3的元素的指针&v[3]。接着我们向这个vector添加另外一些元素并试图复用前面所保存的指针，那么元素v[3]的地址发生了变化，问题是当我们再添加一些新元素时，现有的元素可能会移动到完全不同的位置。
代码如下：
```C++
vector<int> v;

int *old=&v[3];
cout<<"old elment "<< *old <<endl;
cout<<" old address "<< old <<endl;

cout<< "Adding elements ..."<<endl;
for(int i=0;i<10;i++)
    v.push_back(i*10);

cout<<"new elment "<< *old <<endl;
cout<<" new address "<< &v[3] <<endl;

//Output如下：

```

> **解释**：创建一个vector时，它默认分配一定数量的元素（默认16），接着当我们试图添加超出容量的元素时，该vector就会分配一个新的更大的数组，把原先的元素从旧位置复制到新位置，然后继续添加新元素，直到新的容量也被用完。
旧的内存被销毁，可以用于其他用途。但同时指针仍指向旧的位置，如果我们继续操作该指针，比如写入某值到该位置上，它不会改变对应元素的值（该值已位于别处），而这块内存做了其他用途，这样的后果非常不妙，可能会导致程序崩溃。

如果是引用呢，也会发生同样的事情。因为引用知道一个变量的地址，但为了访问它所指向的内存，并不需要在变量前加上星号，语法不同，结果却是一样的。

迭代器也一样，如果我们保存vector中迭代器的任意元素，它可能会在vector的内容被修改之后失效，因为对应的迭代器可能被移动到其他位置。
因此在修改vector之前所得到的指向其中某个元素的任何指针、引用或迭代器在vector由于增加元素而被修改之后就不应该再使用。由于STL库的整体精神是一种容器替换另一种容器，而原先的代码没有问题，那么几乎所有STL容器及所有可能修改容器长度的操作（增删元素）行为都一样：容器被修改后，它的迭代器不再有效。

## Chapter 7 内存泄漏
第一章我们就讲到内存泄露，这是一个很重要的问题，作者分析了几个例子，更全面的定义了内存泄露：如果我们分配了内存（new操作符），必须由某对象负责：

> * 删除这块内存 
> * 使用正确的delete操作符 
> * 该任务只执行一次
> * 在完成了对该内存的使用后，应尽快执行此项任务 

删除内存的责任即称为对象的所有权，所以总结出来：内存泄露是由于被分配的内存的所有权丢失了。这个问题的解决方案是当我们分配新内存时，必须立即把指向这块内存的指针赋值给某个智能指针，这样妈妈再也不用担心删除这块内存的问题了，该任务之后完全由智能指针负责，那关于智能指针我们需要注意些什么呢？
> * 是否允许对智能指针进行复制 
> * 如果是，在智能指针的多份拷贝中，到底哪一个负责删除它们共同指向的对象？ 
> * 智能指针是否表示指向一个对象数组或对象的指针
> * 智能指针是否对应于一个常量指针和非常量指针 

取决于这些问题的答案，我们有多种不同的智能指针，C++社区中有stl的auto_ptr和boost库的shared_ptr，后者是更值得使用的。这里有两类智能指针，能够满足前面讨论的所有需要，有效的防止内存泄露。这两种指针的不同之处在与 引用计数指针可以被复制，而作用域指针不能被复制，但作用域指针效率更高，下面具体讨论它 
**1.引用计数指针**
引用计数指针可以被复制，因此一个智能指针的几份拷贝可以指向同一个对象，这就产生了由哪份拷贝负责删除它们共同指向的对象这个问题。答案是这组智能指针中最后消亡的那个将删除它所指向的对象（类似于最后一个屋子的人负责关灯的概念）。
其中这些指针共享一个计数器，记录有多少个智能指针引用同一个对象。这表明当有人创建一个指向目标对象的智能指针的一份拷贝是，计数加1，反之任何智能指针删除时，计数减1，同时我们也意识到该指针不是线程安全的并且创建计数指针的实参开销比较大。
```C++
//scpp_vector.h
#ifndef  _SCPP_VECTOR_
#define  _SCPP_VECTOR_

#include <vector>
#include "scpp_assert.h"

namespace scpp {

//wrapper around std::vector,在[]提供了临时的安全检查：重载[] <<运算符
template<typename T>
class vector : public std::vector<T> {
	public:
		 typedef unsigned size_type;

		 //常用的构造函数 commonly use cons
		 explicit vector(size_type n=0) : std::vector<T>(n) {}
		 
		 vector(size_type n,const T& value) : std::vector<T>(n,value) {}

		 template <class InputIterator> vector(InputIterator first,InputIterator last) 
		 	: std::vector<T>(first,last) {}
		 	
		 //Note : we don't provide a copy-cons and assignment operator  ?

		//使用scpp::vector提供更安全的下标访问实现，它可以捕捉越界访问错误
		 T& operator[] (size_type index) {
		 	SCPP_ASSERT( index < std::vector<T>::size() ,
		 			   "Index " << index << " must be less than " << std::vector<T>::size());
		 	return std::vector<T>::operator[](index);
		 }

		 //? difference 
		 const T& operator[] (size_type index) const {
		 	SCPP_ASSERT( index < std::vector<T>::size() ,
		 			"Index " << index << " must be less than " << std::vector<T>::size());
		 	return std::vector<T>::operator[](index);
		 }

		 	//允许此函数访问这个类的私有数据
		 	//friend std::ostream& operator<< (std::ostream& os,const ) ?
		};
} //namespace scpp

template<typename T>
inline  std::ostream& operator<< (std::ostream& os,const scpp::vector<T>& v) {
	for(unsigned i=0 ;i<v.size();i++) {
			os << v[i];
			if( i+1 < v.size()) os << " ";
	}
	return os;
}
#endif



```
**2.作用域指针**
当我们并不打算复制智能指针，只是想保证被分配的资源被正确的回收，可采用一个更简单的方法-作用域指针
```C++
#ifndef _SCPP_SCOPEDPTR_H_
#define _SCPP_SCOPEDPTR_H_

#include "scpp_assert.h"

namespace scpp {

template<typename T>
class ScopedPtr {
    public:

    	explicit ScopedPtr(T *p=NULL) :m_ptr(p) {}

    	ScopedPtr<T>& operator=(T *p) {
    		if(m_ptr !=p) {
    			delete m_ptr;
    			m_ptr=p;
    		}
    		return *this;
    	}

    	~ScopedPtr() {
    		delete m_ptr;
    	}

    	T* get() const {
    		return m_ptr;
    	}

    	T* operator->() const {
			SCPP_ASSERT(m_ptr!=NULL,"Attempt to use operator -> on NULL pointer.");
			return m_ptr;
		}

		T& operator* () const {
			SCPP_ASSERT(m_ptr!=NULL,"Attempt to use operator * on NULL pointer.");
			return *m_ptr;
		}

		//把对象的所有权释放给原始调用者
		T* release() {
			T *p=m_ptr;
			m_ptr=NULL;
			return p;
		}



	private:
		T *m_ptr;

		//copy is forbidden
		ScopedPtr(const ScopedPtr<T>& rhs);
		ScopedPtr<T>& operator=(const  ScopedPtr<T>& rhs);

};
} //namespace scpp

#endif

//test_vector.cpp
#include "scpp_vector.h"
#include <iostream>

using namespace std;
int main() {
//usage-创建一个具有指定数量的vector：scpp::vector<int> v(n); 把n个vector元素都初始化为一个值：scpp::vector<int> v(n，val)
//方法3：scpp::vector<int> v; v.reserve(n),表示开始的vector是空的，对应的size()为0,
//并且开始添加元素时，在长度达到n之前，不会出现导致速度降低的容量增长现象
	scpp::vector<int> vec;
    for(int i=0;i<3;i++){
    	vec.push_back(4*i);
	}
	cout << "The vector is : "<< vec <<endl;

	for(int i=0;i<=vec.size();i++) {
		cout << "Value of vector at index " << i << " is " << vec[i] <<endl;
	}
	return 0;
}
```
该类最重要的属性就是析构函数删除它指向的对象，并且作用域指针不能被复制（因为都声明为了私有，任何试图复制这种指针的代码都无法通过编译，这样就消除了对指向同一个对象的同一个智能指针的多份拷贝计数的需要）
**3.用智能指针实行所有权**
用法：
```C++
//不需要调用者负责删除这个对象，而是让函数返回一个智能指针
//更多的依赖编译器而不是程序员
RefCountPtr<A> res(new A(inputs));

ScopedPtr<A> res; //创建一个空作用域指针
void B::create(const Input& ins,ScopedPtr<A>& res);
//创建一个NULL值的作用域指针，并用下面的方法填充它，
//这种方法也不会使该函数创建的对象所有权出现错误
```
**4.解引用NULL指针**
对于解引用NULL指针，一般只需要在声明后或者分配内存后判断下指针是否为NULL指针即可，作者的办法只是更为抽象而已，这里就不再描述




## Chapter 8 避免在析构函数中编写代码
作者有以下几点看法：
需要编写析构函数的原因可能有好几个：
> * 在基类中，可能需要声明虚析构函数，这样就可以使用一个指向基类的指针指向一个派生类的实例
> * 在派生类中，为了增加可读性，可以声明为虚析构函数
> * 可能需要声明析构函数并不抛出任何异常

讨论第三个情形，C++文化中认为从析构函数中抛出异常是不好的思路，因而析构函数本身是在一个异常已经抛出的情况下被调用的，在这个过程中再次抛出一个异常会core dump的，因而析构函数声明为空是不会抛出任何异常的,如`virtual ~A() throw() {}`

它认为如果编写了析构函数，最好让析构函数为空，下面的做法是接受的：
`virtual ~MyClass() {} //空代码` 

下面我们讨论为什么析构函数应该是空的？
1 如果使用智能指针，我们根本就不需要使用析构函数，因为编译器自动会为我们自动生成一个析构函数完成这些任务，在减少工作量的同时，也减少了出现脆弱代码的可能性
2 程序中的错误可能会抛出一个异常，所以如果构造函数抛出异常了呢？
解释：C++社区一般认为从构造函数抛出异常是一种潜在的危险行为，因为我们试图在堆栈创建一个对象，如果正常完成了任务，表明最后析构函数被调用，但是如果构造函数出现了问题呢，它抛出了异常，这意味着对应的析构函数将不会被调用，就导致了内存泄漏。尽管存在意见分歧，但允许构造函数抛出异常是有很充分的理由的，因为构造函数没有返回值，如果有些输入值是错误的，比如为空字符串，我们该怎么做呢，所以允许抛出异常，只要使析构函数是空，这种做法是可以接受的。(可以设计一个小试验判断哪些析构函数会被调用？哪些不会被调用：设计类C继承类A，并含有类B对象b,如果C的构造函数抛出一个异常`throw "Don't like C";`，会发生什么情况呢？结果只有C的析构函数没有执行，AB的析构函数均被调用)

最后总结一下，避免在析构函数中写代码规则如下：
> * 使用智能指针，它会自动完成初始化和调用析构函数，执行清理工作
> * 从构造函数抛出异常时避免内存泄漏应使类析构函数保持为空函数

