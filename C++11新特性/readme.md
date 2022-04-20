[TOC]

# C++ 11 的新特性

## 类型推导
### auto类型推导
自动类型推导，编译器会在**编译期间**通过初始值推导出变量的类型，通过 auto 定义的变量必须有初始值
```c++
auto var = val1 + val2; 
```
### decltype类型推导
decltype 是“declare type”的缩写，译为“声明类型”。和 auto 的功能一样，都用来在**编译时期**进行自动类型推导。如果希望**从表达式中推断出要定义的变量的类型，但是不想用该表达式的值初始化变量**，这时就不能再用 auto。decltype 作用是选择并返回操作数的数据类型
```c++
decltype(val1 + val2) var1 = 0; 
```
### 区别
- auto 根据 = 右边的初始值 val1 + val2 推导出变量的类型，并将该初始值赋值给变量 var；decltype 根据 val1 + val2 表达式推导出变量的类型，变量的初始值和与表达式的值无关
- auto 要求变量必须初始化，因为它是根据初始化的值推导出变量的类型，而 decltype 不要求，定义变量的时候可初始化也可以不初始化

## lambda表达式

```c++
[capture list] (parameter list) -> return type
{
   function body;
};
```
- capture list：捕获列表，指 lambda 所在函数中定义的局部变量的列表
    - \[ ]: 空捕获列表
    - \[变量名, 变量名, ...]: 对所在函数中**指定变量名的变量**采用值捕获(默认, 如果变量名前加了&, 则是引用捕获)
    - \[&]: 隐式捕获,  对所在函数中**所有变量**采用**引用捕获**
    - \[=]: 隐式捕获, 对所在函数中**所有变量**采用**值捕获**
    - \[&, 变量名,变量名,...]: 未指定变量名的变量采用引用捕获, 指定变量名的采用值捕获
    - \[=, 变量名,变量名,...]:未指定变量名的变量采用值捕获, 指定变量名的采用引用捕获


### lambda的主要应用场景
  - 一两个地方使用的简单操作
  - 某些库函数只接受一元谓词和二元谓词,lambda的捕获列表使得它能使用更多的参数
    - 例子: 从find_if()说起:
        ```c++
        InputIterator find_if ( InputIterator first, InputIterator last, Predicate pred )它在区间[first,end)  //中搜寻使一元判断式pred为true的第一个元素。如果没找到，返回end。但是pred只接受一元谓词(只接受一个参数),

        int sz=6;

        find_if(str.begin(),str.end(),[sz](const string& a){return a.size()>=sz;}) //此时可以使用lamba

        bool chack_size(const string& s, string::size_type sz){
            return s.size()>=sz;
        }
        find_if(words.begin(), words.end(), bind(chack_size(),_1, sz)) //也可以使用bind函数
        ```

## 右值引用
绑定到右值的引用，用 && 来获得右值引用，右值引用只能绑定到要销毁的对象。为了和右值引用区分开，常规的引用称为左值引用。
```c++
#include <iostream>
#include <vector>
using namespace std;
int main()
{
    int var = 42;
    int &l_var = var;
    int &&r_var = var; // error: cannot bind rvalue reference of type 'int&&' to lvalue of type 'int' 错误：不能将右值引用绑定到左值上

    int &&r_var2 = var + 40; // 正确：将 r_var2 绑定到求和结果上
    return 0;
}
```
std::move 可以将一个左值强制转化为右值，继而可以通过右值引用使用该值，以用于移动语义
## move()的实现原理
函数原型:
```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
	return static_cast<typename remove_reference<T>::type &&>(t);
}
```
引用折叠原理:
- 右值传递给上述函数的形参 T&& 依然是右值，即 T&& && 相当于 T&&。
- 左值传递给上述函数的形参 T&& 依然是左值，即 T&& & 相当于 T&。

通过引用折叠原理可以知道，move() 函数的形参既可以是左值也可以是右值

remove_reference 具体实现：
```c++
//原始的，最通用的版本
template <typename T> struct remove_reference{
    typedef T type;  //定义 T 的类型别名为 type
};
 
//部分版本特例化，将用于左值引用和右值引用
template <class T> struct remove_reference<T&> //左值引用
{ typedef T type; }
 
template <class T> struct remove_reference<T&&> //右值引用
{ typedef T type; }   
  
//举例如下,下列定义的a、b、c三个变量都是int类型
int i;
remove_refrence<decltype(42)>::type a;             //使用原版本，
remove_refrence<decltype(i)>::type  b;             //左值引用特例版本
remove_refrence<decltype(std::move(i))>::type  b;  //右值引用特例版本 
```

转换过程:
```c++
int var = 10; 

转化过程：
1. std::move(var) => std::move(int&& &) => 折叠后 std::move(int&)

2. 此时：T 的类型为 int&，typename remove_reference<T>::type 为 int，这里使用 remove_reference 的左值引用的特例化版本

3. 通过 static_cast 将 int& 强制转换为 int&&

整个std::move被实例化如下
int && move(int& t) 
{
    return static_cast<int&&>(t); 
}
```

std::move() 实现原理总结：
- 利用引用折叠原理将右值经过 T&& 传递类型保持不变还是右值，而左值经过 T&&
变为普通的左值引用，以保证模板可以传递任意实参，且保持类型不变；
- 然后通过 remove_refrence 移除引用，得到具体的类型 T；
- 最后通过 static_cast<> 进行强制类型转换，返回 T&& 右值引用。



## delete和default函数
- delete 函数：= delete 表示该函数不能被调用
- default 函数：= default 表示编译器生成默认的函数，例如：生成默认的构造函数

## inline 函数
用于定义内联函数。内联函数，像普通函数一样被调用，但是在调用时并不通过函数调用的机制而是直接在调用点处展开，这样可以大大减少由函数调用带来的开销，从而提高程序的运行效率。

### 工作原理
- 内联函数不是在调用时发生控制转移关系，而是在**编译阶段将函数体嵌入到每一个调用该函数的语句块中**，编译器会将程序中出现内联函数的调用表达式用内联函数的函数体来替换。

- 普通函数是将程序执行转移到被调用函数所存放的内存地址，当函数执行完后，返回到执行此函数前的地方。转移操作需要保护现场，被调函数执行完后，再恢复现场，该过程需要较大的资源开销。

- 内联函数和宏定义的区别: 内联函数是在编译时展开，而宏在编译预处理时展开, 会对参数的类型、函数体内的语句编写是否正确等进行检查；在编译的时候，内联函数直接被嵌入到目标代码中去，而宏只是一个简单的文本替换。

### 使用方法：
1. 类内定义成员函数默认是内联函数
在类内定义成员函数，可以不用在函数头部加 inline 关键字，因为编译器会自动将类内定义的函数（构造函数、析构函数、普通成员函数等）声明为内联函数，代码如下：
```c++
#include <iostream>
using namespace std;

class A{
public:
    int var;
    A(int tmp){ 
      var = tmp;
    }    
    void fun(){ 
        cout << var << endl;
    }
};

int main()
{    
    return 0;
}
```
2. 类外定义成员函数，若想定义为内联函数，需用关键字声明当在类内声明函数，在类外定义函数时，如果想将该函数定义为内联函数，则可以在类内声明时不加 inline 关键字，而在类外定义函数时加上 inline 关键字。
```c++
#include <iostream>
using namespace std;

class A{
public:
    int var;
    A(int tmp){ 
      var = tmp;
    }    
    void fun();
};

inline void A::fun(){
    cout << var << endl;
}

int main()
{    
    return 0;
}
```

## volatile 

volatile 的作用：当对象的值可能在程序的控制或检测之外被改变时，应该将该对象声明为 violatile，告知编译器不应对这样的对象进行优化。

volatile不具有原子性。

volatile 对编译器的影响：使用该关键字后，编译器不会对相应的对象进行优化，即不会将变量从内存缓存到寄存器中，防止多个线程有可能使用内存中的变量，有可能使用寄存器中的变量，从而导致程序错误

使用 volatile 关键字的场景：
- 当多个线程都会用到某一变量，并且该变量的值有可能发生改变时，需要用 volatile 关键字对该变量进行修饰；
- 中断服务程序中访问的变量或并行设备的硬件寄存器的变量，最好用 volatile 关键字修饰。
- volatile 关键字和 const 关键字可以同时使用，某种类型可以既是 volatile 又是 const ，同时具有二者的属性。

一个volatile的例子：
```C++
const  int const_value = 100;
int * ptr = (int *)&const_value;  
*ptr = 200;
cout<<*ptr<<"  "<<const_value<<endl;
//结果输出为200，100，其实已经通过指针将const_value变量地址内的值已经变成了200，但是由于编译器的优化，将const类型的const_value放在编译器的符号表内，计算时编译器直接从表中取值，所以输出为100

const volatile int const_value = 100;
int * ptr = (int *)&const_value;  
*ptr = 200;
cout<<*ptr<<"  "<<const_value<<endl;
//结果为200，200， 编译器阻止了上述优化
```

## extern C 

当 C++ 程序 需要调用 C 语言编写的函数，C++ 使用链接指示，即 extern"C"指出任意非 C++ 函数所用的语言。

```c++
// 可能出现在 C++ 头文件<cstring>中的链接指示
extern "C"{
    int strcmp(const char*, const char*);
}
```

## 显式类型转换

- static_cast: 用于良性转换，一般不会导致意外发生，风险很低。
- const_cast:	用于 const 与非 const、volatile 与非 volatile 之间的转换。
- reinterpret_cast:	高度危险的转换，这种转换仅仅是对二进制位的重新解释，不会借助已有的转换规则对数据进行调整，但是可以实现最灵活的 C++ 类型转换。
- dynamic_cast:	借助 RTTI（Runtime Type Identification, “运行时类型识别”），用于类型安全的向下转型（Downcasting）。
    - 1.其他三种都是编译时完成的，动态类型转换是在程序运行时处理的，运行时会进行类型检查。
    - 2.只能用于带有虚函数的基类或派生类的指针或者引用对象的转换，转换成功返回指向类型的指针或引用，转换失败返回 NULL；不能用于基本数据类型的转换。
    - 3.在向上进行转换时，即派生类类的指针转换成基类类的指针和 static_cast 效果是一样的，（注意：这里只是改变了指针的类型，指针指向的对象的类型并未发生改变）。

使用方法:`xxx_cast<newType>(data)`


## 模板

模板定义以关键字 template 开始，后跟一个模板参数列表。

```c++
template <typename T, typename U, ...>
```

### 函数模板和类模板:
```c++
#include<iostream>

using namespace std;

template <typename T>
T add_fun(const T & tmp1, const T & tmp2){
    return tmp1 + tmp2;
}

template <typename T>
class Complex
{
public:
    //构造函数
    Complex(T a, T b)
    {
        this->a = a;
        this->b = b;
    }

    //运算符重载
    Complex<T> operator+(Complex &c)
    {
        Complex<T> tmp(this->a + c.a, this->b + c.b);
        cout << tmp.a << " " << tmp.b << endl;
        return tmp;
    }

private:
    T a;
    T b;
};

int main(){
    int var1=10, var2=20;
    add_fun(var1, var2);  // 隐式调用
    add_fun<int>(var1, var2); // 显式调用

    Complex<int> a(10, 20);
    Complex<int> b(20, 30);
    Complex<int> c = a + b;
    return 0;
}
```
函数模板和类模板的区别
- 实例化方式不同：函数模板实例化由编译程序在处理函数调用时自动完成，类模板实例化需要在程序中显式指定。
- 实例化的结果不同：函数模板实例化后是一个函数，类模板实例化后是一个类。
- 默认参数：类模板在模板参数列表中可以有默认参数。
- 特化：函数模板只能全特化；而类模板可以全特化，也可以偏特化。
- 调用方式不同：函数模板可以隐式调用，也可以显式调用；类模板只能显式调用。

### 可变参数模板

接受可变数目参数的模板函数或模板类。将可变数目的参数被称为参数包，包括模板参数包和函数参数包
- 模板参数包：表示零个或多个模板参数；
- 函数参数包：表示零个或多个函数参数。

```c++
template <typename T, typename... Args> // Args 是模板参数包
void foo(const T &t, const Args&... rest); // 可变参数模板，rest 是函数参数包
```
用省略号来指出一个模板参数或函数参数表示一个包，在模板参数列表中，class… 或 typename… 指出接下来的参数表示零个或多个类型的列表；一个类型名后面跟一个省略号表示零个或多个给定类型的非类型参数的列表。当需要知道包中有多少元素时，可以使用 sizeof… 运算符。
```c++
#include <iostream>

using namespace std;

template <typename T>
void print_fun(const T &t)
{
    cout << t << endl; // 最后一个元素
}

template <typename T, typename... Args>
void print_fun(const T &t, const Args &...args)
{
    cout << t << " ";
    print_fun(args...);
}

int main()
{
    print_fun("Hello", "wolrd", "!");
    return 0;
}
/*运行结果：
Hello wolrd !

*/
```
说明：可变参数函数通常是递归的，第一个版本的 print_fun 负责终止递归并打印初始调用中的最后一个实参。第二个版本的 print_fun 是可变参数版本，打印绑定到 t 的实参，并用来调用自身来打印函数参数包中的剩余值。

### 模板特化
模板特化的原因：模板并非对任何模板实参都合适、都能实例化，某些情况下，通用模板的定义对特定类型不合适，可能会编译失败，或者得不到正确的结果。因此，当不希望使用模板版本时，可以定义类或者函数模板的一个特例化版本。

模板特化：模板参数在某种特定类型下的具体实现。分为函数模板特化和类模板特化
- 函数模板特化：将函数模板中的全部类型进行特例化，称为函数模板特化。
- 类模板特化：将类模板中的部分或全部类型进行特例化，称为类模板特化

特化分为全特化和偏特化：
- 全特化：模板中的模板参数全部特例化。
- 偏特化：模板中的模板参数只确定了一部分，剩余部分需要在编译器编译时确定。

```c++
#include <vector>
#include <iostream>
using namespace std;

// 函数模板
template<typename T, class N>
void compare(T num1, N num2) {
    cout << "standard function template" << endl;
}

//全特化
template<>
void compare(int s1,int s2) {
    cout<<"full specialization"<<endl;
}

// 对部分模板参数进行特化
template<class N>
void compare(int num1, N num2) {
    cout<< "partitial specialization" <<endl;
}

// 将模板参数特化为指针(模板参数的部分特性)
template<typename T, class N>
void compare(T* num1, N* num2) {
    cout << "new partitial specialization" << endl;
}

// 将模板参数特化为另一个模板类
template<typename T, class N>
void compare(std::vector<T>& vecLeft, std::vector<T>& vecRight) {
    cout << "to vector partitial specialization" << endl;
}

int main() {
    // 调用非特化版本 compare<int,int>(int num1, int num2)
    string s1="234";
    string s2="456";
    compare(s1,s2);
    // 调用全特化版本
    compare(1,2);
    // 调用偏特化版本 compare<char>(int num1, char num2)
    compare(30,'1');

    int a = 30;
    char c = '1';
    // 调用偏特化版本 compare<int,char>(int* num1, char* num2)
    compare(&a, &c);

    vector<int> vecLeft{0};
    vector<int> vecRight{1,2,3};
    // 调用偏特化版本 compare<int,char>(int* num1, char* num2)
    compare<int,int>(vecLeft,vecRight);
}
/*  运行结果
standard function template
partitial specialization
partitial specialization
new partitial specialization
to vector partitial specialization
*/
```

## 泛型编程如何实现

泛型编程实现的基础：模板。模板是创建类或者函数的蓝图或者说公式，当时用一个 vector 这样的泛型，或者 find 这样的泛型函数时，编译时会转化为特定的类或者函数。

泛型编程涉及到的知识点较广，例如：容器、迭代器、算法等都是泛型编程的实现实例。面试者可选择自己掌握比较扎实的一方面进行展开。

- 容器：涉及到 STL 中的容器，例如：vector、list、map 等，可选其中熟悉底层原理的容器进行展开讲解。
- 迭代器：在无需知道容器底层原理的情况下，遍历容器中的元素。
- 模板：可参考本章节中的模板相关问题。