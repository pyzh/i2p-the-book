# 引用

> 注：本文中提及到的引用，是广义的“引用[^1]”概念，而不仅仅是C++的引用[^2]。

简单一点讲，引用是一种关系，比如，本文中引用了大量的Wikipedia词条，每一个链接对应下来就是一个“引用”。这种链接关系，就可以叫做引用。

这个概念其实很简单，比如我要完成这篇文章，有些事情说不明白的，可能需要参考Wikipedia的解释，这个时候有两种办法，一种是通过引文的方式摘取一段相关的文字，另一种方式就是直接给出链接。

这两种都可能带来一些问题。比如前者，如果引文部分在Wikipedia做了修改，这边是不能够自动做同步的；比如后者，通过链接引用自然不需要专门去对内容修改与否做特别的关注，但是如果所链接的内容直接被删除了，也是一个很糟糕的事情：毕竟原文不在了，又没有引文，这样基本上就没有任何地方可以还原回来你期望引用的内容了。

编程中也有类似的概念与之对应。比如你要取一个序列中的特定位置的元素，一种办法是直接取出来，复制给某一个变量。另外一种就是记录一个位置信息，每次要用的时候再去取。

同样地也有之前我们提到的两个问题：要么，对方更新了你得不到同步；要么双方之间通过一个位置信息做关联，但实际的数据丢失了。


## 复制与引用

### 内存空间

其实我们可以简单地把程序运行的内存空间看做一个由字节组成的特别大的序列。每个变量其实对应到这些序列中特定的连续的一段空间。

比如：

| 序号 | 内容 |
| -- | -- |
| 0xDEADBE00 | DE |
| 0xDEADBE01 | AD |
| 0xDEADBE02 | BE |
| 0xDEADBE03 | EF |

假设这段空间可以用来表示一个32的整数，同时假设是Big Endian[^3]的字节序（Byte Order）。那么这段空间就可以用来表示数字`3735928559`(0xDEADBEEF)，如果是一个长度为4的字符串，那么就是`" ޭ��"`（`"\xDE\xAD\xBE\xEF"`）。

### 复制

在这种情况下，如果我们定义两个变量a和b，其实就是处在内存的不同位置上。当我们选择把a赋值给b的时候，相当于把a所处空间的内容，依次放到b所对应的空间中去。

其实也就相当于把a的内容**复制**了一份给了b。当然，对于一个32位的整数来说，相当于复制了4个字节。对于一个长度为10的32位整数array来说，相当于复制了$$4\times10$$也就是40个字节。对于其他类型的数据当然也是根据其类型和内容来确定复制的字节数量。

当然在引用出现之前，复制做到这么简单易懂是最好的了。

### 指针

试想这样一个结构：一个长度为100，array其元素的类型是一个长度为100的32位整数array。也就是`array<array<int, 100>, 100>`。大致看上去可以当成一个$$100\times100$$的矩阵。

如果这样类型的两个变量之间相互赋值，参与复制的就是$$4\times100\times100$$个字节。假设每次只能复制一个字节的话，一个简单的赋值操作就要进行40000次复制。当然对于实际应用程序来说，这只是一些很小的数据，但也足够引起重视了。

如果我们用前面提到的思路，只通过数据在内存这个超大的序列中的位置以及其类型来表示该数据，这样对于这种大型数据结构，我们就可以节省很多的复制时间和空间占用了。

C++的指针（Pointer）就可以作为这样一种结构来达到我们期望的效果：

```C++
using matrix_t = array<array<int, 100>, 100>; // alias to matrix_t
matrix_t matrix {};

matrix_t* matrix_pointer = &matrix;
```

`using`关键字可以帮我们把`array<array<int, 100>, 100>`做一个简单的重命名，不用每次都写成这样一堆。而`matrix_t*`就是matrix_t类型对应的指针类型。

比如我们希望取到matrix的第一个元素。只需要调用`matrix.at(0)`就好，同样地，利用matrix_pointer也只需要`matrix_pointer->at(0)`就可以了。这两个操作其实经历的是同样的过程，只不过matrix_pointer有一个解引用的过程：要知道这个指针所指向的对象究竟是哪一个。

同样地，如果我们不只是希望通过matrix_pointer来操作matrix，而是希望直接拿到matrix这个对象，C++提供了解引用运算符`*`，也就是`*matrix_pointer`，就是原本matrix对象了。

### 指针的本质

> 对于许多纠结细节的人来说，往往会深入到“指针就是地址”这种无谓的争论中去。其实C++的指针只是一个无法保证内存安全和类型安全的代理对象（参考下文）。无需纠结“内存地址”的概念（当然本书也并未提及）。
> 
> 另外，对于指针，往往又会被称之为**裸指针**（raw pointer），只适合在极限的时候非常需要性能等情况下可能考虑使用。毕竟，由于兼容性的原因，指针默认允许一些算术运算、不会对指向空间的有效性做检查，允许`void*`类型的指针做任意指针类型之间的转换。这其中的任意一项都是非常损害程序安全性和可靠性的。

指针的引入给我们带来的很多方便，但其本质其实非常的简单。

首先，作为一个对象，它自身是没有任何特殊的能力的，只有当指向相应类型的对象的时候，才能作为该对象的一个引用者存在，也就是说，可以用来间接地表示该对象。

其次，除了赋值、类型转换和指针算术之外任何操作，其实最终都是由指针所指向的对象来来响应。指针只是起了一个中间人的作用。

像指针这种形式的对象随处可见（比如下文提到的智能指针），当然有些也许会在“引用”之外，做一些额外的事情，但本质上都是作为一个对象的**代理**（proxy[^4]）而存在。这种设计模式可以很大的提高程序设计时的灵活性，比如，在子类型多态中，指针能够根据对象的实际类型做动态分发。

### 引用

前面提到了指针的诸多缺点，在C++中，另有一个“引用”的概念，可以在某种程度上弥补指针的不足。

首先，为了避免指针的内存安全问题，C++引用必须显式初始化，也就是说，C++引用一存在就会绑定一个对象；同时，初始化以后，引用关系不能再改变。

其次，C++引用是强类型的，也就避免了C语言中各种隐晦的类型转换可能造成的安全问题。

另外，C++引用也可以看成一个高级的指针与语法糖[^5]。

```C++
matrix_t matrix {};
matrix_t& matrix_ref = matrix;

matrix_ref[0] //Just like matrix[0]
```
由于C++引用可以在某些方面表现上完全跟原本的对象一致，有些C++教材也会误把引用当成一种**别名**的形式。如：

> 引用（reference）是c++对c语言的重要扩充。引用就是某一变量（目标）的一个别名，对引用的操作与对变量直接操作完全一样。
>  —— 百度百科
>  
> 条款5 引用是别名而非指针  —— *C++ Common Knowledge: Essential Intermediate Programming*（《C++必知必会》）

对于入门理解“引用”的概念，这种比喻可以理解，但是C++的引用之复杂远非一个“别名”所能描述。

另外，由于C++自身对于对象的生命周期检查不足，即使是使用引用也会导致一些明显的问题，例如：
```C++
std::string& f()
{
    std::string s = "Example";
    return s; // exits the scope of s:
              // its destructor is called and its storage deallocated
}
 
std::string& r = f(); // dangling reference
std::cout << r;       // undefined behavior: reads from a dangling reference
std::string s = f();  // undefined behavior: copy-initializes from a dangling reference
```

因为C++会自动管理变量的生命周期，在f函数返回时，局部变量s已经被自动销毁并释放内存空间了，这种时候所获取到的s的引用就是悬空引用（dangling reference[6]）。

尽管大多数现代的C++编译器都可以检测出这种错误，但仍然不能够解决如何返回函数在局部构建的对象的引用这个问题。C++11以后引入的智能指针，就轻松的搞定了这一点。

### 智能指针

智能指针有很多种，我们先来看看shared_ptr。

上节的程序其使用了shared_ptr的版本如下：

```C++
#include <memory>

std::shared_ptr<std::string> f()
{
    auto s = make_shared<string>("Example");
    return s;
}
 
std::shared_ptr<std::string> r = f();
std::cout << *r;
std::string s = *f();
```
智能指针，当然就是比普通的“指针”更智能了。其中非常重要的一个点就是能够自动管理对象的生命周期。shared_ptr允许一个对象被多个指针所持有，只有当所有智能指针都被正常释放以后，才会自动释放该对象的资源。

我们来看上面代码中的f，在其中构造了一个字符串，同时是作为一个智能指针构造的。在f返回的时候，智能指针s被复制作为返回值，然后作为f的局部变量被释放。这个时候，f的返回值（比如变量r）的还是存活的，所以f中构造的字符串继续存活，直到r被正常释放。

shared_ptr通过引用计数的方式来管理对象的生命周期，也即通过统计引用者的数量来决定资源是否需要释放。这种方式简单高效，配合C++能够自动管理对象生命周期的特性，提供了一种灵活而又简便的引用机制。

## 引用的复杂性

我们之前说过，如果没有引用之前，复制非常简单直观。当引入了引用以后，右给我们带来哪些复杂度呢？

### 再谈复制

前面提到，复制其实可以简单到只是一块块地把字节放到另外一个地方就可以了。当然如果是raw pointer的话，我们也可以简单地这样做：
```c++
int x = 10;
int* y = &x;
int* z = y;
```
本质上也只是把y里面几个字节的内容复制到了z那儿。

但是我们说了，raw pointer是靠不住的：要么资源泄漏了，要么指针悬空了。所以我们才请smart pointer来帮忙。
```C++
auto x = make_shared<int>(10);  // Construct resource & assigned to x
auto y = x;  // Ref to resource, and increase the ref count
```
对于智能指针来说，要么，需要转移所有权，要么需要做引用计数的改变。这个时候复制的过程其实包含了一大堆的检查和处理：这其实也可以理解为啥C++要有Copy Constructor和Copy Assignment Operator了。

还有一点也是很重要的，当然这种事情通常发生在默认采用引用语义的语言中：如果对一个层次比较深的复杂的对象做拷贝，结果得到的很有可能不是你期待的结果。

比如下面的Java代码：
```java
class VeryComplexObject {    
    SomeObject blablabla;
    
    ...
    public VeryComplexObject clone() {
        VeryComplexObject vco = new ...;
        vco.setBlablabla(this.getBlablabla());
        ...
    }
}

VeryComplexObject vco = VeryComplexObjectBuilder().build();
VeryComplexObject vco2 = vco;
VeryComplexObject vco3 = vco.clone();
```
在Java中，所有的非基本类型的变量都是特定对象实例的引用。所以他们之间相互赋值的结果其实跟C++中raw pointer赋值没什么太多本质上的区别。所以vco2其实跟vco一模一样没有什么改动。

那vco3就有什么不同了吗？没有。

表面上看似乎创建了一个新的对象然后把vco所有的属性都重新设置给了新的对象。但还是因为本质上是引用语义的原因，vco和vco3的blablabla属性还是对应的同一个对象实例。也就是说：
```java
vco3.getBlablabla().setFoo(new Foo());
```
这句代码同样地会影响vco的blablabla属性的foo属性。

这就是Java中常提到的Shallow Copy和Deep Copy的问题，这种只是把属性重新set一遍的做法就是所谓的Shallow Copy（**浅拷贝**）。于是最终也引出了一个Cloneable和obj.clone()方法，作为一个约定来进行Deep Copy。跟C++的Copy Ctor和Copy Assignment Operator也算是殊途同归吧。

### 循环引用

这也是一个比较糟心的问题。
先看代码吧。
```c++
struct Node {
    int value;
    shared_ptr<Node> next;
};
void f() {
    auto first = make_shared<Node>(1, null_ptr);
    auto second = make_shared<Node>(2, first);
    first->next = second;
}

f();
```
我们构造了两个Node对象（下一节可以看到，这是单链表典型的结构）。其中，first的next属性引用了second，而second的next属性又引用回了first。

这样就构成了一个环。也就是循环引用（Circular Reference）的名字的来源。

在这段代码执行完函数f返回的时候，first和second两个变量的生命周期就结束了，于是这两个智能指针会被释放和回收，他们所引用的资源的引用计数也会减一，然后……

没有然后了。

对应的两个Node对象不会被回收。因为两个对象之间相互被引用着，他们各自的引用计数都没有恢复成0，所以无法被回收。于是这两个尴尬的对象就又成为了被泄漏的内存资源。

### 自动资源管理



[^1]: https://en.wikipedia.org/wiki/Reference_(computer_science)
[^2]: http://en.cppreference.com/w/cpp/language/reference
[^3]: https://en.wikipedia.org/wiki/Endianness
[^4]: https://en.wikipedia.org/wiki/Proxy_pattern
[^5]: 《C++语言的设计与演化》
[^6]: https://en.wikipedia.org/wiki/Dangling_reference


