### 深入C++回调



##### 一、什么是回调函数？

- 可执行代码作为参数传入到其他可执行代码
- 并且是其他的可执行代码执行这段可执行代码。
  - 客户端：负责实现这段可执行代码，但是不知道什么时候调用这段代码。
  - 服务端：负责调用这段可执行代码，但是不知道这段代码的具体实现。

```C++
A() {
  // output P
}

B(fn) {
  fn();  // B knows only fn, not A
         // B treats fn as a variable
}

B(A);  // B called at T
       // B calling fn() (i.e. calling A())
```

我们可以看到

- 函数A作为参数fn传入函数B
  - B不知道A的存在，只知道fn。
  - B将fn嘴鸥为一个传入函数的局部变量处理。
- 函数B通过fn的语法，调用函数A
  - B在调用时刻T，调用fn。
  - B调用fn可能是为了得到结果P。

##### 二、C语言的回调

###### 2.1、同步回调

​	**同步回调**：把函数b传递给函数a。执行a的时候，回调了b，a要一直等到b执行完才能继续执行；

​	我们看一下数组从小到大排序的实现代码。代码调用了函数qsort进行排序，并通过指定的compare为参数，实现元素的大小的比较。

```C++
int compare (const void * a, const void * b) {
    return (*(int*) a - *(int*) b);
}

...

int values[] = { 20, 10, 50, 30, 60, 40 };
qsort (values, sizeof (values) / sizeof (int), sizeof(int), compare);
```

​	我们在调用compare的时刻T是在调用qsort结束之前(qsort并没有返回)，这样子的回调被称为同步回调。

###### 2.2、异步回调

​	**异步回调**：把函数b传递给函数a。执行a的时候，回调了b，然后a就继续往后执行，b独自执行。创建一个单独线程执行回调函数。	



```C++
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>
#include "A.h"
//-----------------------底层实现A-----------------------------
typedef struct parameter{
　　int a ;
　　pcb callback;
}parameter;
  
void* callback_thread(void *p1)//此处用的是一个线程
{
　　//do something
　　parameter* p = (parameter*)p1 ;
　　sleep(5);
　　p->callback(p->a);//函数指针执行函数，这个函数来自于应用层B
}
  
//留给应用层B的接口函数
void SetCallBackFun(int a, pcb callback)
{
　　printf("A:start\n");
　　parameter *p = malloc(sizeof(parameter)) ;
　　p->a = a;								 /// 同样将参数也注册也
　　p->callback = callback;  /// 将回调函数注册给p的的回调函数 如果可以在异步线程中调用
  
　　//创建线程
　　pthread_t pid;
　　pthread_create(&pid,NULL,callback_thread,(void *) p);
　　printf("A:end\n");
  
　　//阻塞，等待线程pid结束，才往下走
　　pthread_join(pid,NULL);
}

//-----------------------应用者B-------------------------------
void fCallBack(int a) // 应用者增加的函数，此函数会在A中被执行
{
　　//do something
　　printf("B:start\n");
　　printf("a = %d\n",a);
　　sleep(5);
　　printf("B:end\n");
}
  
int main(void)
{
　　SetCallBackFun(4,fCallBack);
　　return 0;
}
```

##### 三、函数式闭包

##### 四、深入C++回调函数

​	从语言上看，回调是一个调用函数的过程，涉及两个角色：计算和数据。其中，回调的计算是一个函数，而回调的数据来源于两部分：

-  **绑定** *(bound)* 的数据，即回调的 **上下文**
-  **未绑定** *(unbound)* 的数据，即执行回调时需要额外传入的数据

捕获了上下文的回调函数就成为了闭包，即 **闭包 = 函数 + 上下文**。在面向对象的语言中：

- 闭包 一般通过对象实现
- 上下文 一般作为闭包对象的数据成员 和闭包属于关联、组合、聚合等关系

从对象所有权的角度看，上下文又可以分为：

- 不变的上下文：
  - 数值/字符串/结构体 等基本类型，永远 **不会失效**
  - 使用时，一般 **不需要考虑** 生命周期问题
- **弱引用** *(weak reference)* 上下文（**可变** *(mutable)* 上下文）
  - 闭包 **不拥有** 上下文，所以回调执行时 **上下文可能失效**
  - 如果使用前没有检查，可能会导致 **崩溃**
- **强引用** *(strong reference)* 上下文（**可变** *(mutable)* 上下文）
  - 闭包 **拥有** 上下文，能保证回调执行时 **上下文一直有效**
  - 如果使用后忘记释放，可能会导致 **泄漏**

###### 4.1、回调是同步还是异步

​	同步回调在构造闭包的调用栈里局部执行。	

``` C++
int total = 0;
std::for_each(std::begin(scores), std::end(scores),
              [&total](auto score) { total += score; });
            // ^ context variable |total| is always valid
```

- 绑定的数据：total 局部变量的上下文 弱引用 所有权在闭包外
- 未绑定的数据 ：Score 每次迭代传递的值	



​	异步回调在构造后存储起来，在未来某个时刻非局部执行。用户界面不阻塞UI线程响应用户输入，在后台线程异步加载背景图片，加载完成后在从UI线程显示到界面。

```C++
// callback code
void View::LoadImageCallback(const Image& image) {
  // WARNING: |this| may be invalid now!
  if (background_image_view_)
    background_image_view_->SetImage(image);
}

// client code
FetchImageAsync(
    filename,
    base::Bind(&View::LoadImageCallback, this));
```

- 绑定的数据：base::Bind绑定了View的this指针
- 未绑定的数据：View::LoadImageCallback` 的参数 `const Image& image



**回调时上下文会不会失效**

由于闭包没有 **弱引用上下文** 的所有权，所以上下文可能失效：

- 对于 **同步回调**，上下文的 **生命周期往往比闭包长**，一般不失效
- 而在 **异步回调** 调用时，上下文可能已经失效了

- 传递普通对象的 **裸指针**，容易导致悬垂引用
- 传递捕获了上下文的 lambda 表达式，**无法检查** lambda 表达式捕获的 **弱引用** 的 **有效性**

**C++ 核心指南** *(C++ Core Guidelines)* 也有类似的讨论：

- [F.52: Prefer capturing by reference in lambdas that will be used locally, including passed to algorithms](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-reference-capture)
- [F.53: Avoid capturing by reference in lambdas that will be used nonlocally, including returned, stored on the heap, or passed to another thread](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-value-capture)



**如何处理失效的（弱引用）上下文**

如果弱引用上下文失效，回调应该 **及时取消**。例如 异步加载图片 的代码，可以给 `base::Bind` 传递 `View` 对象的 **弱引用指针**，即 `base::WeakPtr<View>`：

```C++
FetchImageAsync(
    filename,
    base::Bind(&View::LoadImageCallback, AsWeakPtr()));
 // use |WeakPtr| rather than raw |this| ^
}
```

在执行 `View::LoadImageCallback` 时：

- 如果界面还在显示 View对象仍然有效，则会执行Image::SetImage显示背景图片
- 如果界面不再显示，此时传入的弱引用失效，不会在执行回调。

> - base::WeakPtr 属于侵入式智能指针，非线程安全
> - base::Bind针对base::WeakPtr扩展了base::IsWeakReceiver<>的检测，调用前可以判断弱引用的有效性
> - 也可以基于std::weak_ptr非侵入式表示弱引用所有权，但是和std::shared_ptr”绑定“，在使用前需要调用lock。（[弱回调 |《当析构函数遇到多线程 —— C++ 中线程安全的对象回调》陈硕](https://github.com/downloads/chenshuo/documents/dtor_meets_mt.pdf)）

###### 4.2、回调执行一次OR多次

我们可以先看下C语言中的回调函数

- 没有闭包的概念，所以需要函数或者程序员手动管理上下文的声明周期，即申请/释放上下文
- 由于资源的所有权不明确，难以判断指针T*是强引用还是弱引用。

我们看一段C语言回调代码：

```C++
// callback code
void do_send(evutil_socket_t fd, short events, void* context) {
  char* buffer = (char*)context;
  // ... send |buffer| via |fd|
  free(buffer);  // free |buffer| here!
}

// client code
char* buffer = malloc(buffer_size);  // alloc |buffer| here!
// ... fill |buffer|
event_new(event_base, fd, EV_WRITE, do_send, buffer);
```



****

**何时销毁上下文**

对于面向对象的回调，强引用上下文的 **所有权属于闭包**。例如，改写 异步/非阻塞发送数据 的代码：

```C++
using Event::calllback = base::OnceCallback<void()>;

// callback code
void DoSendOnce(std::unique_ptr<Buffer> buffer) {
  // ...
}  // free |buffer| via |~unique_ptr()|

// client code
std::unique_ptr<Buffer> buffer = ...;
event->SetCallback(base::BindOnce(&DoSendOnce,
                                  std::move(buffer)));
```

- 我们在构造闭包函数：buffer移动到base::OnceCallback内部。通过std::move()
- 回调执行的时候：buffer从OnceCallback移动到DoSendOnce()的参数中，并在回调结束后销毁。（**所有权转移**，`DoSendOnce` **销毁 强引用参数**）
- 闭包销毁时：如果回调没有执行，`buffer` 未被销毁，则此时销毁（**保证销毁且只销毁一次**）



```C++
using Event::calllback = base::RepeatingCallback<void()>;

// callback code
void DoSendRepeating(const Buffer* buffer) {
  // ...
}  // DON'T free reusable |buffer|

// client code
Buffer* buffer = ...;
event->SetCallback(base::BindRepeating(&DoSendRepeating,
                                       base::Owned(buffer)));
```

- 构造闭包时：`buffer` **移动到** `base::RepeatingCallback` 内
- 回调执行时：每次传递 `buffer` 指针，`DoSendRepeating` **只使用** `buffer` 的数据（`DoSendRepeating` **不销毁 弱引用参数**）
- 闭包销毁时：总是由闭包销毁 `buffer`（**有且只有一处销毁的地方**）

****

**何时传递（强引用）上下文**

根据 [可拷贝性](https://bot-man-jl.github.io/articles/?post=2018/Resource-Management#资源和对象的映射关系)，强引用上下文又分为两类：

- 不可拷贝的 **互斥所有权** *(exclusive ownership)*，例如 `std::unique_ptr`
- 可拷贝的 **共享所有权** *(shared ownership)*，例如 `std::shared_ptr`

STL 原生的 `std::bind`/`lambda` + `std::function` 不能完整支持 **互斥所有权** 语义

```C++
// OK, pass |std::unique_ptr| by move construction
auto unique_lambda = [p = std::unique_ptr<int>{new int}]() {};

// OK, pass |std::unique_ptr| by ref
unique_lambda();

// Bad, require |unique_lambda| copyable
std::function<void()>{std::move(unique_lambda)};

// OK, pass |std::unique_ptr| by move
auto unique_bind = std::bind([](std::unique_ptr<int>) {},std::unique_ptr<int>{});

// Bad, failed to copy construct |std::unique_ptr|
unique_bind();

// Bad, require |unique_bind| copyable
std::function<void()>{std::move(unique_bind)};
```

- unique_lambda/unique_bind
  - 只能移动，不能拷贝
  - 不能构造 `std::function`
- `unique_lambda` 可以执行，上下文在 `lambda` 函数体内作为引用
- `unique_bind` 不能执行，因为函数的接收参数要求拷贝 `std::unique_ptr`



STL 回调在处理 **共享所有权** 时，会导致多余的拷贝：

```C++
auto shared_lambda = [p = std::shared_ptr<int>{}]() {};
std::function<void()>{shared_lambda};  // OK, copyable

auto shared_func = [](std::shared_ptr<int> ptr) {     // (6)
  assert(ptr.use_count() == 6);
};
auto p = std::shared_ptr<int>{new int};               // (1)
auto shared_bind = std::bind(shared_func, p);         // (2)
auto copy_bind = shared_bind;                         // (3)
auto shared_fn = std::function<void()>{shared_bind};  // (4)
auto copy_fn = shared_fn;                             // (5)
assert(p.use_count() == 5);
```

- shared_lambda/shared_bind
  - 可以拷贝，对其拷贝也会拷贝闭包拥有的上下文
  - 可以构造 `std::function`
- `shared_lambda` 和对应的 `std::function` 可以执行，上下文在 `lambda` 函数体内作为引用
- `shared_bind` 和对应的 `std::function` 可以执行，上下文会拷贝成新的 `std::shared_ptr`