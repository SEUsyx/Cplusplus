[TOC]

# 内存管理

## C/C++ 内存分区：
![](../编译/pictures/内存分段模型.png)
- 栈(stack):   存放函数的局部变量、函数参数、返回地址等，由编译器自动分配和释放。
- 堆(heap):   动态申请的内存空间，就是由 malloc 分配的内存块，由程序员控制它的分配和释放，如果程序执行结束还没有释放，操作系统会自动回收
- 全局/静态存储区(.bss 段和 .data 段):  存放全局变量和静态变量，程序运行结束操作系统自动释放，在 C 语言中，未初始化的放在.bss段中，初始化的放在data 段中，C++ 中不再区分
- 常量存储区(.data 段):存放的是常量，不允许修改，程序运行结束自动释放。
- 代码区(.text 段)：存放代码，不允许修改，但可以执行。编译后的二进制文件存放在这里

## 栈和堆的区别
- 申请方式：栈是系统自动分配，堆是程序员主动申请。
- 申请后系统响应：分配栈空间，如果剩余空间大于申请空间则分配成功，否则分配失败栈溢出；申请堆空间，堆在内存中呈现的方式类似于链表（记录空闲地址空间的链表），在链表上寻找第一个大于申请空间的节点分配给程序，将该节点从链表中删除，大多数系统中该块空间的首地址存放的是本次分配空间的大小，便于释放，将该块空间上的剩余空间再次连接在空闲链表上。
- 栈在内存中是连续的一块空间（向低地址扩展）最大容量是系统预定好的，堆在内存中的空间（向高地址扩展）是不连续的。
- 申请效率：栈是有系统自动分配，申请效率高，但程序员无法控制；堆是由程序员主动申请，效率低，使用起来方便但是容易产生碎片。
- 存放的内容：栈中存放的是局部变量，函数的参数；堆中存放的内容由程序员控制

## C++的自由存储区
什么是自由存储区: **由new和delete管理的区域,是一个逻辑概念, 可以在堆区,全局区或其他区域**  
由malloc/free 申请或者销毁的内存分布在堆区，由new/delete申请的内存在自由存储区。但是new/delete在默认情况下是使用了malloc/free 那我们能不能说自由存储区是堆区的子集呢?不能, 在C++中我们可以对new进行重载（实际上new 和 delete 有一种形式不能被重载，之后会谈到），我们可以让对象的内存空间不在堆区而在全局区或者其他(布局new)。这个时候自由存储区就不是堆区的子集。

## new

new operator ：new操作符
例子：`A * a=new A;`
- 执行过程
  - 调用operator new（new 函数）分配内存，operator new (sizeof(A)) 
  - 调用构造函数生成类对象，A::A() 
  - 返回相应指针 
- 不能被重载

operator new ：new函数
有三种形式：
```C++
void* operator new (std::size_t size) throw (std::bad_alloc);
void* operator new (std::size_t size, const std::nothrow_t& nothrow_constant) throw();
void* operator new (std::size_t size, void* ptr) throw();
```
- 第一种分配size个字节的存储空间，并将对象类型进行内存对齐。如果成功，返回一个非空的指针指向首地址。失败抛出bad_alloc异常。
  - 调用方法：`A* a = new A;`
  - 可以被用户重载
- 第二种在分配失败时不抛出异常，它返回一个NULL指针。
  - 调用方法：`A* a = new(std::nothrow) A`
  - 可以被用户重载
- 第三种是placement new版本，它本质上是对operator new的重载，
  - 调用方法：`A* a =new (p) A()`
  - 不可以被用户重载
  - 与其说定位放置new操作是申请空间，还不如说是利用已经请好的空间，真正的申请空间的工作是在此之前完成的。 
  - 定位生成对象时，会自动调用类A的构造函数，但是由于对象的空间不会自动释放（对象实际上是借用别人的空间），所以必须显示的调用类的析构函数， 
  - 作用：创建对象(调用该类的构造函数)但是不分配内存，而是在已有的内存块上面创建对象。用于需要反复创建并删除的对象上，可以降低分配释放内存的性能消耗。
  - 应用场景举例：在一个server中对于客户端的请求，每个客户端的每一次上行数据我们都需要为此申请一块内存，当我们处理完请求给客户端下行回复时释放掉该内存，表面上看者符合c++的内存管理要求，没有什么错误，但是仔细想想很不合理，为什么我们每个请求都要重新申请一块内存呢，要知道每一次内存的申请，系统都要在内存中找到一块合适大小的连续的内存空间，这个过程是很慢的（相对而言），极端情况下，如果当前系统中有大量的内存碎片，并且我们申请的空间很大，甚至有可能失败。为什么我们不能共用一块我们事先准备好的内存呢？可以的，我们可以使用placement new来构造对象，那么就会在我们指定的内存空间中构造对象。 

## delete 
delete 的实现原理：
1. 首先执行该对象所属类的析构函数；
2. 进而通过调用 operator delete 的标准库函数来释放所占的内存空间。

delete 和 delete [] 的区别：
- delete 用来释放单个对象所占的空间，只会调用一次析构函数；
- delete [] 用来释放数组空间，会对数组中的每个成员都调用一次析构函数。

## new和malloc的区别
- 申请内存的位置：new操作符从自由存储区（free store）上为对象动态分配内存空间，而malloc函数从堆上动态分配内存。
- new/delete是C++关键字，需要编译器支持。malloc/free是库函数，需要头文件支持。
- 使用new操作符申请内存分配时无须指定内存块的大小，编译器会根据类型信息自行计算。而malloc则需要显式地指出所需内存的尺寸。
- new操作符内存分配成功时，返回的是对象类型的指针，类型严格与对象匹配，无须进行类型转换，故new是符合类型安全性的操作符；而malloc内存分配成功则是返回void * ，需要通过强制类型转换将void*指针转换成我们需要的类型。
- malloc是面向内存的，你要开多大，就给你开多大，开了就不管了。new是面向对象的，根据你指定的数据类型来申请对应的空间，并且能够直接内部调用构造函数生成对象。
- new 分配失败时，会抛出 bad_alloc 异常，malloc 分配失败时返回空指针。

## malloc 的原理:
- 当开辟的空间小于 128K 时，调用 brk() 函数，通过移动 _enddata 来实现；
    - brk() 函数实现原理：向高地址的方向移动指向数据段的高地址的指针 _enddata。
- 当开辟空间大于 128K 时，调用 mmap() 函数，通过在虚拟地址空间中开辟一块内存空间来实现。
    - mmap 内存映射原理：
        - 进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域；
        - 调用内核空间的系统调用函数 mmap()，实现文件物理地址和进程虚拟地址的一一映射关系；
        - 进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存（主存）的拷贝。

# 变量存储模型

c++管理数据内存的三种方式：自动存储、静态存储
c++11新增了第四种：线程存储（变量的持续性和整个线程一样）

| 存储描述  | 持续性  | 作用域 | 链接性 | 如何声明    | 限定符   | 内存空间|
| :------ | :----| :----- | :----- | :------- | :---- |:---|
| 自动存储(**局部变量**)   | 自动   | 代码块 | 无     | 在代码块中  | auto(c++ 11 不再支持)  |栈|
| 寄存器 (**局部变量**)  | 自动  | 代码块 | 无     | 在代码块中   | register   |栈|
| 静态,无链接性(**静态局部变量**)    | 静态              | 代码块 | 无     | 在代码块中，使用关键字static     | static  | 静态存储区|
| 静态，内部链接性(**静态全局变量**) | 静态              | 文件   | 内部   | 不在任何函数内，使用关键字static | static                                 | 静态存储区|
| 静态，外部链接性(**全局变量**)     | 静态              | 文件   | 外部   | 不在任何函数内                   | extern（声明时可以省略，使用是必须有） |静态存储区|
| 动态存储    | 由new和delete控制 | 无     | 无     | 函数内外都可以声明    |      |自由存储区|

注：const的全局变量的链接性为内部，即外部不可调用，可使用extern关键字来覆盖默认的内部链接性变成外部链接性

## 变量描述:
- 全局变量：具有全局作用域。全局变量只需在一个源文件中定义，就可以作用于所有的源文件。当然，其他不包含全局变量定义的源文件需要用extern 关键字再次声明这个全局变量。
- 静态全局变量：具有文件作用域。它与全局变量的区别在于如果程序包含多个文件的话，它作用于定义它的文件里，不能作用到其它文件里，即被static 关键字修饰过的变量具有文件作用域。这样即使两个不同的源文件都定义了相同名字的静态全局变量，它们也是不同的变量。
- 局部变量：具有局部作用域。它是自动对象（auto），在程序运行期间不是一直存在，而是只在函数执行期间存在，函数的一次调用执行结束后，变量被撤销，其所占用的内存也被收回。
- 静态局部变量：具有局部作用域。它只被初始化一次，自从第一次被初始化直到程序运行结束都一直存在，它和全局变量的区别在于全局变量对所有的函数都是可见的，而静态局部变量只对定义自己的函数体始终可见。

## 全局变量定义在头文件中有什么问题？
如果在头文件中定义全局变量，当该头文件被多个文件 include 时，该头文件中的全局变量就会被定义多次，导致重复定义，因此不能再头文件中定义全局变量。

## 存储说明符

- auto ：声明变量为自动变量
- register：声明变量为寄存器变量
- static：静态存储变量
- extern：外部链接性
- thread_local：线程变量
- mutable：可以修改const限定的结构中的变量
- volatile：不让编译器执行优化（例如发现几条语句中两次使用了某个变量，编译器不让程序查找这个值两次，会将这个值缓存到寄存器中，这中间可能导致变量的值发生变化） 
 
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

## 内存对齐

编译器将程序中的每个“数据单元”安排在字的整数倍的地址指向的内存之中

### 内存对齐的原则:
- 结构体变量的首地址能够被其最宽基本类型成员大小与对齐基数中的较小者所整除；
- 结构体每个成员相对于结构体首地址的偏移量 （offset）都是该成员大小与对齐基数中的较小者的整数倍，如有需要编译器会在成员之间加上填充字节 （internal padding）；
- 结构体的总大小为结构体最宽基本类型成员大小与对齐基数中的较小者的整数倍，如有需要编译器会在最末一个成员之后加上填充字节（trailing padding）

### 内存对齐的原因(主要是硬件设备方面的问题)
  - 某些硬件设备只能存取对齐数据，存取非对齐的数据可能会引发异常；
  - 某些硬件设备不能保证在存取非对齐数据的时候的操作是原子操作；
  - 相比于存取对齐的数据，存取非对齐的数据需要花费更多的时间；
  - 某些处理器虽然支持非对齐数据的访问，但会引发对齐陷阱（alignmenttrap）；
  - 某些硬件设备只支持简单数据指令非对齐存取，不支持复杂数据指令的非对齐存取。

### 内存对齐的优点
- 便于在不同的平台之间进行移植，因为有些硬件平台不能够支持任意地址的数据访问，只能在某些地址处取某些特定的数据，否则会抛出异常；
- 提高内存的访问效率，因为 CPU 在读取内存时，是一块一块的读取。

## 内存泄漏
由于疏忽或错误导致的程序未能释放已经不再使用的内存
- 并非指内存从物理上消失，而是指程序在运行过程中，由于疏忽或错误而失去了对该内存的控制，从而造成了内存的浪费。
- 常指堆内存泄漏，因为堆是动态分配的，而且是用户来控制的，如果使用不当，会产生内存泄漏。
- 使用 malloc、calloc、realloc、new 等分配内存时，使用完后要调用相应的 free 或 delete释放内存，否则这块内存就会造成内存泄漏。
- 指针重新赋值, 之前指向的内存空间无法找到

### 防止内存泄漏
- 内部封装：将内存的分配和释放封装到类中，在构造的时候申请内存，析构的时候释放内存。（说明：但这样做并不是最佳的做法，在类的对象复制时，程序会出现同一块内存空间释放两次的情况）
- 智能指针：智能指针是 C++ 中已经对内存泄漏封装好了一个工具，可以直接拿来使用

### 内存泄漏的检测工具
Valgrind:   一款用于内存调试、内存泄漏检测以及性能分析的软件开发工具  

对内存泄漏的检测包括以下方面
- 对未初始化内存的使用；
- 读/写释放后的内存块；
- 读/写超出malloc分配的内存块；
- 读/写不适当的栈中内存块；
- 内存泄漏，指向一块内存的指针永远丢失；
- 不正确的malloc/free或new/delete匹配；
- memcpy()相关函数中的dst和src指针重叠

使用方法: 
1.不保存日志:
` valgrind --leak-check=full --show-reachable=yes --track-origins=yes  -v ./可执行文件 参数`
--leak-check=full指的是完全检查内存泄漏
--show-reachable=yes是显示内存泄漏的地点
--track-origins=yes查看未初始化的来源
2.保存日志:
`valgrind --tool=memcheck --leak-check=full --show-reachable=yes --track-origins=yes  --log-file=memchk.log -v ./可执行文件 参数`

# 智能指针
智能指针是为了解决动态内存分配时带来的内存泄漏以及多次释放同一块内存空间而提出的。C++11 中封装在了 < memory > 头文件中

C++11 中智能指针包括以下三种：
  - 共享指针（shared_ptr）：资源可以被多个指针共享，使用计数机制表明资源被几个指针共享。通过 use_count() 查看资源的所有者的个数，可以通过 unique_ptr、weak_ptr 来构造，调用 release() 释放资源的所有权，计数减一，当计数减为 0 时，会自动释放内存空间，从而避免了内存泄漏。
  - 独占指针（unique_ptr）：独享所有权的智能指针，资源只能被一个指针占有，该指针不能拷贝构造和赋值。但可以进行移动构造和移动赋值构造（调用move() 函数），即一个 unique_ptr 对象赋值给另一个 unique_ptr 对象，可以通过该方法进行赋值。
  - 弱指针（weak_ptr）：指向 shared_ptr 指向的对象，能够解决由shared_ptr带来的循环引用问题。


## 实现一个共享指针:(计数原理)
```c++
#include <iostream>
#include <memory>
#include <assert.h>

using namespace std;

template <typename T>
class SmartPtr
{
private :
    T *_ptr; 
    size_t *_count;  //计数(指针类型, 这样指向同一对象的共享指针的计数值就是一样的)

public:
    SmartPtr(T *ptr = nullptr) : _ptr(ptr)
    {
        if (_ptr)
        {
            _count = new size_t(1);
        }
        else
        {
            _count = new size_t(0);
        }
    }

    ~SmartPtr()
    {
        (*this->_count)--;
        if (*this->_count == 0)
        {
            delete this->_ptr;
            delete this->_count;
        }
    }

    SmartPtr(const SmartPtr &ptr) // 拷贝构造：计数 +1
    {
        if (this != &ptr)
        {
            this->_ptr = ptr._ptr;
            this->_count = ptr._count;
            (*this->_count)++;
        }
    }

    SmartPtr &operator=(const SmartPtr &ptr) // 赋值运算符重载
    {
        if (this->_ptr == ptr._ptr)
        {
            return *this;
        }
        if (this->_ptr) // 将当前的 ptr 指向的原来的空间的计数 -1
        {
            (*this->_count)--;
            if (this->_count == 0)
            {
                delete this->_ptr;
                delete this->_count;
            }
        }
        this->_ptr = ptr._ptr;
        this->_count = ptr._count;
        (*this->_count)++; // 此时 ptr 指向了新赋值的空间，该空间的计数 +1
        return *this;
    }

    T &  operator*()
    {
         //assert(this->_ptr==nullptr); //不能正确运行(不管_ptr是不是null,都会执行assert), 改成下面的就好了
        assert(this->_ptr);
        return *(this->_ptr);
    }

    T *operator->()
    {
        assert(this->_ptr );
        return this->_ptr;
    }

    size_t use_count()
    {
        return *this->_count;
    }
};
class testa{
public:
    int a;
    int b;

    testa(int a_=0,int b_=0):a(a_),b(b_){}
    ~testa(){cout<<"~test"<<endl;}
};
int main(){

    testa* p=new testa(5,10);
    if(p==nullptr) cout<<"???"<<endl;
    assert(p);
    SmartPtr<testa> p1(new testa(5,10));
    SmartPtr<testa> p2;
    //{
        //p1 = p;
        cout << p1.use_count() << "  "<<p1->a<< endl;
        p2 = p1;
        //cout << p1.use_count() << "  "<<(*p1).a << endl;
        cout << p2.use_count() << "  "<<p2->a << endl;
    //}
    cout << p1.use_count() << "  " << endl;
    cout << p2.use_count() << "  " << endl;

    return 0;
}
```

## 循环引用问题:
在如下例子中定义了两个类 Parent、Child，在两个类中分别定义另一个类的对象的共享指针，由于在程序结束后，两个指针相互指向对方的内存空间，导致内存无法释放。该被调用的析构函数没有被调用，从而出现了内存泄漏。

解决办法：weak_ptr
- weak_ptr 对被 shared_ptr 管理的对象存在非拥有性（弱）引用，在访问所引用的对象前必须先转化为 shared_ptr；
- weak_ptr 用来打断 shared_ptr 所管理对象的循环引用问题，若这种环被孤立（没有指向环中的外部共享指针），shared_ptr 引用计数无法抵达 0，内存被泄露；令环中的指针之一为弱指针可以避免该情况；
- weak_ptr 用来表达临时所有权的概念，当某个对象只有存在时才需要被访问，而且随时可能被他人删除，可以用 weak_ptr 跟踪该对象；需要获得所有权时将其转化为 shared_ptr，此时如果原来的 shared_ptr 被销毁，则该对象的生命期被延长至这个临时的 shared_ptr 同样被销毁
```c++
#include <iostream>
#include <memory>

using namespace std;

class Child;
class Parent;

class Parent {
private:
//    shared_ptr<Child> ChildPtr;   //造成循环引用
    weak_ptr<Child> ChildPtr; //解决办法
public:
    void setChild(shared_ptr<Child> child) {
        this->ChildPtr = child;
    }

    void doSomething() {
        if (this->ChildPtr.use_count()) {
            cout<<"parent do something"<<endl;
        }
    }

    ~Parent() {
        cout<<"~parent"<<endl;
    }
};

class Child {
private:
    shared_ptr<Parent> ParentPtr;
public:
    void setPartent(shared_ptr<Parent> parent) {
        this->ParentPtr = parent;
    }
    void doSomething() {
        if (this->ParentPtr.use_count()) {
            cout<<"child do something"<<endl;
        }
    }
    ~Child() {
        cout<<"~child"<<endl;
    }
};

int main() {
    weak_ptr<Parent> wpp;
    weak_ptr<Child> wpc;
    {
        shared_ptr<Parent> p(new Parent);
        shared_ptr<Child> c(new Child);
        cout<<"here1"<<endl;
        cout << p.use_count() << endl; // 2
        cout << c.use_count() << endl; // 2
        p->setChild(c);
        cout<<"here2"<<endl;
        cout << p.use_count() << endl; // 2
        cout << c.use_count() << endl; // 2
        c->setPartent(p);
        cout<<"here3"<<endl;
        cout << p.use_count() << endl; // 2
        cout << c.use_count() << endl; // 2
        wpp = p;
        wpc = c;
    }
    //结束后没有没销毁, 因为计数为1
    cout << wpp.use_count() << endl;  // 1
    cout << wpc.use_count() << endl;  // 1
    return 0;
}
```
```
引用成环的运行结果:
here1
1
1
here2
1
2
here3
2
2
1
1

解决后的运行结果:
here1
1
1
here2
1
1
here3
2
1
~child
~parent
0
0
```




