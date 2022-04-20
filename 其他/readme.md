- [如何判断结构体是否相等？能否用 memcmp 函数判断结构体相等？](#如何判断结构体是否相等能否用-memcmp-函数判断结构体相等)
- [指针和引用的区别](#指针和引用的区别)
- [参数传递时，值传递、引用传递、指针传递的区别](#参数传递时值传递引用传递指针传递的区别)

## 如何判断结构体是否相等？能否用 memcmp 函数判断结构体相等？

需要重载操作符 == 判断两个结构体是否相等，不能用函数 memcmp 来判断两个结构体是否相等，因为 memcmp 函数是逐个字节进行比较的，而结构体存在内存空间中保存时存在字节对齐，字节对齐时补的字节内容是随机的，会产生垃圾值，所以无法比较。

```c++
#include <iostream>

using namespace std;

struct A
{
    char c;
    int val;
    A(char c_tmp, int tmp) : c(c_tmp), val(tmp) {}

    friend bool operator==(const A &tmp1, const A &tmp2); //  友元运算符重载函数
};

bool operator==(const A &tmp1, const A &tmp2)
{
    return (tmp1.c == tmp2.c && tmp1.val == tmp2.val);
}

int main()
{
    A ex1('a', 90), ex2('b', 80);
    if (ex1 == ex2)
        cout << "ex1 == ex2" << endl;
    else
        cout << "ex1 != ex2" << endl; // 输出
    return 0;
}
```

## 指针和引用的区别
- 指针所指向的内存空间在程序运行过程中可以改变，而引用所绑定的对象一旦绑定就不能改变。（是否可变）
- 指针本身在内存中占有内存空间，引用相当于变量的别名，在内存中不占内存空间。（是否占内存）
- 指针可以为空，但是引用必须绑定对象。（是否可为空）
- 指针可以有多级，但是引用只能一级。（是否能为多级）


## 参数传递时，值传递、引用传递、指针传递的区别
- 值传递：形参是实参的拷贝，函数对形参的所有操作不会影响实参。
- 指针传递：本质上是值传递，只不过拷贝的是指针的值，拷贝之后，实参和形参是不同的指针，通过指针可以间接的访问指针所指向的对象，从而可以修改它所指对象的值。
- 引用传递：当形参是引用类型时，我们说它对应的实参被引用传递。

```c++
#include <iostream>
using namespace std;

void fun1(int tmp){ // 值传递
    cout << &tmp << endl;
}

void fun2(int * tmp){ // 指针传递
    cout << tmp << endl;
}

void fun3(int &tmp){ // 引用传递
    cout << &tmp << endl;
}

int main()
{
    int var = 5;
    cout << "var 在主函数中的地址：" << &var << endl;

    cout << "var 值传递时的地址：";
    fun1(var);

    cout << "var 指针传递时的地址：";
    fun2(&var);

    cout << "var 引用传递时的地址：";
    fun3(var);
    return 0;
}

/*
运行结果：
var 在主函数中的地址：0x23fe4c
var 值传递时的地址：0x23fe20
var 指针传递时的地址：0x23fe4c
var 引用传递时的地址：0x23fe4c
*/
```

