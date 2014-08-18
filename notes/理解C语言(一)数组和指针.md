# 理解C语言(一) 数组与指针
## 介绍
人们在使用数组时经常会把等同于指针，并自然而然地假定在所有的情况下数组和指针都是等同的。
为什么出现这样的混淆呢，因为我们在使用时经常可以看到大量的作为函数参数的数组和指针，在这种情况下它是可以互换的，但是人们容易忽视它只是发生在一个特定的上下文环境中，例如printf(“arr at location %x holds string %s”,arr, arr),在这条语句中因printf是函数，数组名既可以当地址，又可以当一个字符数组。
我们也习惯了在 main函数的参数中有这样的char **argv 或char *argv[]的形式，因为argv是一个函数的参数，它诱使我们错误地总结出指针和数组是等价的。
再例如下面一个程序arr.c：
```
#include <stdio.h>
int len(int arr[]){
	return sizeof(arr);
}

int main(){
	int arr[]={1,2,3,4,5};
	printf("%d\n",sizeof(arr)); //sizeof计算类型或变量或类的存储字节大小
    printf("%d\n\n",len(arr));//?同样使用sizeof,为什么结果不同
	
	printf("arr=%p\n",(void *)arr); //指针表示法取地址：arr+0..len-1
	printf("&arr[0]=%p\n\n",(void *)&arr[0]); //数组表示法取地址: &arr[0..len-1]

	int *p=(int *)(&arr+1);//&arr+1与arr+1为什么有区别
    int *p1=(int *)(arr+1);
	printf("arr+1=%p\n",(void *)p1);
    printf("&arr+1=%p\n\n",(void *)p);
     return 0;
}
```
结果如图所示：
![图1](/home/charles/Pictures/2a.png)
可以看出我们使用sizeof计算数组的时候数组名有时候当指针来看，有时候又当整个数组来看待；而数组表示法又和指针表示法等价，数组名前加一个&运算符，它却又不等同于指针的使用。
也就是说，数组和指针并不全都相同。那么数组什么时候等同于指针，什么时候不等同于指针呢

## 数组和指针入门
### 1. 区分定义和声明
extern声明说明编译器对象的类型和名字，描述了其他地方的创建对象
定义要求为对象分配内存；定义指针时编译器并不为指针所指向的对象分配空间，它只是分配指针本身的空间。
```
int a[100];
extern int a[]; //正确并且无需提供关于数组长度的信息
extern int *a;//错误,人们总是错误地认为数组和指针非常类似
```
### 2. 数组和指针是如何访问的
X=Y：左值在编译时可知，表示存储结果的地址;右值表示Y的内容
也就是说编译器为每个变量分配一个左值，该地址在编译时可知，而变量在运行时一直保存于这个地址，而右值只有在运行时才可知。如需用到变量中存储的值，编译器就发出指令从指定地址读入变量值并将它存入寄存器中。
如果编译器需要一个地址，可能要加上偏移量来执行某种操作，它就可以直接进行操作，并不需要增加指令取得具体的地址。相反对于指针，必须先在运行时取得它的值然后才能对它解引用

下面分别是对数组下标的引用和对指针的引用的描述：
```
char a[9]=”abcdefgh”; c=a[i];
```
编译器符号表具有一个地址9980,运行时

步骤1：取i的值，将它与9980相加
步骤2：取地址（9980+i）的内容
```
char *p; c=*p;
```
编译器符号表有一个符号p,它的地址为4624,运行时

步骤1：先得到地址p 的内容，即5081
步骤2：将5081作为字符的地址并取得它的内容

指针的访问明显灵活很多，但需要增加一次额外的提取
### 3. 数组和指针的引用
-  定义为指针，但以数组方式引用

指针定义编译器会告诉你这是一个指向字符的指针，相反数组定义则告诉你是一个字符序列。
下面是对指针进行下标访问的描述：
```
char *p=”hello”;c=p[i];
```
步骤1： 先取符号表中p的地址，提取出其内容
步骤2： 将内容与下标偏移量相加产生一个地址
步骤3： 访问上面那个地址，取得字符


-  定义为数组名，但以指针方式引用


1. **数组名变量代表了数组中第一个元素的地址。它并不是一个指针但却表现得像一个不能被修改的常指针 。因而它不能被赋值**
1. **对数组下标的引用总是可以写成一个指向数组起始地址的指针加上偏移量：**
```
int a[100];
 p=a; p[i];
 p=a;*(p+i); //
```
通常情况下，数组下标是在指针的基础上，所以优化器可以把它转化为更有效率的指针表达形式，并生成相同的机器指令，所以C 语言采用指针形式就是因为指针和偏移量是底层硬件所使用的基本模型，但在处理一维数组时指针见不得比数组更快

1. **为什么要把数组作为函数的参数传递当作指针**
把作为形参的数组和指针等同起来是出于效率的考虑，数组名自动改写成指向数组第一个元素的指针形式，而不是整个数组的拷贝，并且如果要操作数组的元素千万不要在数组名上进行操作，形式应如下:
```
char *test(char a[]) {
	char *p=a;
}
```
在C语言规定中，所有非数组形式的数据均以传值形式（即对实参做一份拷贝并传递给调用的函数，函数不能修改作为实参的实际变量的值而只能修改它的那份拷贝）。
因而有些人喜欢把它理解成数组和函数是传址调用，缺省情况下都是传值调用，数据也可以传址调用，即加&地址运算符，这样传递给函数的是实参的地址而不是实参的拷贝。但**严格意义上传址调用也不十分准确，因为编译器的机制是在被调用的函数中，你拥有的是一个指向变量的指针，而不是变量本身，传递的参数只是指针变量值本身的拷贝。
传值调用的拷贝是指分配了栈上的空间地址，内容和实参值一样而形参的地址肯定与实参地址不一样，因而当指针作为函数参数，你只需要测试指针变量值的实参和形参地址是否不一样就可以知道传递的究竟是指针变量值本身的副本还是该指针指向的变量的副本。**

```
#include <stdio.h>
#include <stdlib.h>

void f2(int *a){
        printf("执行函数f2(a)：\n");
	printf("形参变量a的地址=%p\n",&a);
	printf("形参变量a的值=%p\n\n",a);
        *a=15;
}

int main(){
      int *a=(int *)malloc(sizeof(int));
      *a=10;
      printf("previous *a=%d\n",*a);
      printf("实参变量a的地址%p\n",&a);
      printf("实参变量a的值%p\n\n",a);
      f2(a);
      printf("after *a=%d\n",*a);
      printf("存储*a变量的地址%p\n",a);
      printf("存储指针变量a的地址%p\n",&a);
      return 0;
}
```
结果如下：![图2](/home/charles/Pictures/agg.png)

### 4.  总结：数组和指针的异同点
| 指针 | 数组 |
|--------|--------|
|  保存数据的地址      |     保存数据   |
|  间接访问数据，首先取得指针的内容，把它当做地址，加上偏移量作为新地址提取数据      |   直接访问数据，以数组名+偏移量访问数据     |
|  通常用于动态数据结构，如malloc free      |  通常用于固定数目的元素      |
|   指向匿名数据（指针操作匿名内存）     |     自身即为数据名   |
|   指针始终就是指针，不可改写成数组     |   数组作为函数参数会当做指针看待     |
|     可用下标形式访问指针，一般都是指针作为函数参数，但你要知道实际传递给函数的是一个数组   |  &nbsp;      |
但注意也有例外, 就是把数组当做一个整体来考虑，而对数组的引用不能作为指向该数组第一个元素的指针来代替，看参见介绍中程序arr.c的执行结果：
①　数组作为sizeof的操作数，显示要求的是整个数组的大小，但注意当数组作为函数形参时，自动退化为指针，在函数内部计算sizeof,结果只是计算指针类型的大小，这一般与机器字长有关，两者并不矛盾
通常可以在头文件定义一个宏语句：#define TABLESIZE(arr) (sizeof(arr)/sizeof(arr[0]))
②　使用&获取字符数组的地址
③　数组是一个字符串常量初始值

## 多维数组与指针
注：当提到C语言中的数组时，就把它看做一个向量，或者是某种对象的一维数组，数组的元素也可以是另一个数组。因而多维数组可以看成数组的数组
- ###   数组的内存布局与定位

数组下标的规则告诉我们元素的存储和引用都是线性排列在内存中的，若要计算pea[i][j],  则是要先找到pea[i]的位置，再根据偏移量取得字符，因而pea[i][j]解析为：
*(*(pea+i)+j)
例如:
![图3](/home/charles/Pictures/ag.png)

- ###   初始化二维字符串数组的办法

利用指针数组初始化字符串常量如下`char *p[]={“1heh”,”ghh”,...};`而其他非字符串类型的指针数组不能直接初始化，它的定义如下
```
int r1[]={3,4,5};
int r2[]={0,9,8,4,3};
int r3[]={0};
int *weight[]={r1,r2,r3};
```
上面这种长度不一的数组，我们称之为锯齿状数组。如何创建它，也有技巧：
如，尽量不要选择拷贝整个字符串，拷贝指针即可
```
 char *ip[len];
 char hello[]=”world”;
 ip[i]=&hello[0];  //共享字符串，直接使用现有的
 ip[j]=malloc(strlen(hello)+1);
 strcpy(ip[j],hello); //拷贝字符串，通过分配内存创建一份现有字符串的新鲜拷贝，仅传递指针
```
还有如，在指针数组的末尾增加一个NULL指针，该NULL指针使函数在搜索这个表格时能够检测到表的结束，而无需预先知道表的长度
如查询C源文件中关键字的个数：
```
const char *keyword[]={"do","for",...,NULL};
```
-  ### 多维数组作为参数是如何传递的

当多维数组作为参数时，数组的数组是被改写为数组的指针的，而不是指针的指针，因而

|实参 | 匹配的形式参数 |
|--------|--------|
|  数组的数组char c[8][10]      |  数组的指针 char (*p)[10]      |
|   指针数组 char *c[15]     |  指针的指针 char **c      |
|   数组指针（行指针）char (*c)[64]     |  不改变      |
|    指针的指针    |   不改变     |

因而你之所以在main函数中看到char **argv这样的参数，是因为argv是个指针数组char *argv[]；因而若是声明为数组的数组，它将被编译器改写为char (*argv)[15]

## 实现动态数组
动态数组的数据结构定义为：
```
typedef int T;
typedef struct {
	 T *data;
	 int size; //the size of data
	 int index;
} array_t;
```
动态数组常见的接口函数设计：
```
array_t *array_create(int capacity);
array_t *array_create_default();
void array_push(array_t *arr, T elt);
int array_isempty(array_t *arr);
void array_clear(array_t *arr);
T array_pop(array_t *arr);
void array_set(array_t *arr,T elt,int pos);
T array_get(array_t *arr,int pos);
void array_for_each(array_t *arr);
void array_sort(array_t *arr); //快速排序动态数组
```
具体代码如下：
arraylist.h : [头文件实现](https://github.com/charlesxiong/data_structure/blob/master/arraylist.h)
```
#ifndef _ARRAY_H_
#define _ARRAY_H_
#include <stdio.h>
#include <stdlib.h>
#define DEFAULT_SZ 16
typedef int T;
typedef struct {
	 T *data;
	 int size; //the size of data
	 int index; 
} array_t;

array_t *array_create(int capacity);
array_t *array_create_default();
void array_push(array_t *arr, T elt);
int array_isempty(array_t *arr);
void array_clear(array_t *arr);
T array_pop(array_t *arr);
void array_set(array_t *arr,T elt,int pos);
T array_get(array_t *arr,int pos);
void array_for_each(array_t *arr);
void array_sort(array_t *arr);

static inline array_t *
array_init(int capacity){
	array_t *arr=(array_t *)malloc(sizeof(array_t));
	if(NULL==arr) {
		printf("Allocating memory failed...\n");
		return NULL;
	}
	arr->index=0;
	arr->size=capacity;
	return arr;
} 

static inline void 
swap(T *v,int i,int j){
	T tmp=v[i];
	v[i]=v[j];
	v[j]=tmp;
}

static inline int 
numcmp(T *v1,T *v2){
	if(*v1<*v2)
		return -1;
	else if (*v1 > *v2)
		return 1;
	else
		return 0;
}

static inline void
array_qsort(array_t *arr,int left,int right,int (*comp)(T *,T *)){
	if(left >=right) return;
	int i,last;
	swap(arr->data,left,(left+right)>>1);
	last=left;
	for(i=left+1;i<=right;i++){
		if((*comp)(&arr->data[i],&arr->data[left])<0) //pointer to a function
			swap(arr->data,++last,i);
	}
	swap(arr->data,left,last);
	array_qsort(arr,left,last-1,comp);
	array_qsort(arr,last+1,right,comp);
}
#endif
```
arraylist.c : [动态数组接口实现](https://github.com/charlesxiong/data_structure/blob/master/arraylist.c)
```
#include "arraylist.h"


array_t *array_create(int capacity){
	array_t *arr=array_init(capacity);
	arr->data=malloc(sizeof(T)*arr->size);
	if(NULL==arr->data) {
		printf("Allocating data failed...");
		return NULL;
	}
	return arr;
}

array_t *array_create_default(){
	return array_create(DEFAULT_SZ);
}

void array_push(array_t *arr, T elt) {
	if(arr->index==arr->size) { //increasing to 2*size
		/*method 1: manually copy data */
		int size=arr->size*2;
		T *new_data=(T *)malloc(sizeof(T)*size);
		if(NULL==new_data) {
			printf("Memory allocation error...");
			return ;
	    }
	    printf("Allocating success...\n");
	    T *src=arr->data;
	    T *dest=new_data;
	    int n=arr->size;
	    while(n--){
	    	*dest++=*src++;
	    }
	    free(arr->data);
	    arr->data=new_data;
	    arr->size=size;


	    /*method 2: using realloc*/
	    // arr->size *=2;
	    // arr->data=realloc(arr->data,sizeof(int)*arr->size);
        
	    /*method 3: implement my realloc function*/
	}
	arr->data[arr->index]=elt;

	arr->index++;
}

int array_isempty(array_t *arr){
	return (arr->size==0)?TRUE:FALSE;
}

void array_clear(array_t *arr){
	while(--arr->index){
		arr->data[arr->index]=0;
	}
	arr->size=0;
}

T array_pop(array_t *arr) {
	int elt=arr->data[--arr->index];
	arr->size--;
	return elt;
}

void array_set(array_t *arr,T elt,int pos){
	if(pos<0 || pos >=arr->index ) {
		printf("Invalid position");
		return;
	}
	arr->data[pos]=elt;
}

T array_get(array_t *arr,int pos){
	if(pos<0 || pos >=arr->index ) {
		printf("Invalid position");
		exit(1);
	}
	return arr->data[pos];
}


void array_for_each(array_t *arr){
	int i;
	for(i=0;i < arr->index;i++)
		printf("%d ",arr->data[i]);
	printf("\n");
}



void array_sort(array_t *arr) {
	array_qsort(arr,0,arr->index-1,numcmp);
}
```






