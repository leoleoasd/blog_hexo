---
title: C++未定义行为及其危害
date: 2021-04-04 11:13:09
tags:
	- C++
	- 编译器
categories: 教学
---

## 背景

最近在校内部署了我们开发的[EduOJ](http://github.com/eduoj/backend)以供数据结构与算法课程使用。OJ中使用了 clang 编译器以避免gcc编译器造成的坑（如`#include</dev/random>`能卡死编译器，以及某段很短的代码能产生数G的错误日志）。同时，按照惯例，开启了O2优化。

上线后不久，很多同学反映说OJ不好用。我问咋回事儿，同学说代码本地运行都是对的，提交到平台上之后就是错的了。相关代码片段如下：

1. 这段代码运行结果和本地不一样

```cpp
while (head->next != nullptr) {
  if (cnt == m) {
    cout << p->data << ' ';
      
    // del 是代码作者定义的一个函数，里面调用了
    // delete p
    del(p); 
    cnt = 0;
  } else {
    cnt += 1;
    if (p->next != nullptr) {
      p = p->next;
    } else {
      p = head->next;
    }
  }
}

```

可以看到，他在`delete p`后还访问了`p->next`。

2. 这段更离谱：

```cpp
template<typename T> bool ArrayList<T>::append(T const& value) {
  if(_size >= _capacity){ // border check;
    cout<<"SIZE "<<_size<<" CAP "<<_capacity;
    cout << " The List is overflow!" << endl;
    return false;
  }
  _elem[_size++] = value;
}
```

会输出`SIZE 0 CAP 2000 The List is overflow!`。

看到这里，你可能已经急了。明明`size`是`0`，`cap`是`2000`，为什么`if`里的`size >= cap`会成立呢？先别急，接着看下面的代码：

3. 这段代码很难找出错（所以建议跳过去不看）：

```cpp
#include <iostream>
using namespace std;
int main() {
  int n, s, m, j, tempt, q, k, flag;
  int count = 1;
  int data[2020];
  cin >> n >> s >> m;
  tempt = s;
  for (int i = 0; i < n; i++) {
    data[i] = i + 1;
  }
  for (; count <= n; count++) { // 主要看这个for循环
    for (k = 1, q = 0; k <= m; q++) {
      if (data[(tempt + q - 1) % n] != 0)
        k++;
    }
    if (count != n) {
      cout << data[(tempt + q - 2) % n];
      cout << ' ';
    } else {
      cout << data[(tempt + q - 2) % n];
      return 0;
    }
    data[(tempt + q - 2) % n] = 0;
    for (j = 1, flag = 0; flag != 1; j++) {
      if (data[(tempt + q + j - 2) % n] != 0)
        flag = 1;
      else
        flag = 0;
    }
    tempt = (tempt + q + j - 2) % n;
  }
  return 0;
}
```
但是，强大的Clang编译器有各种错误检查。我们编译的时候加上“未定义行为检测器”试试：

```shell
$ clang++ a.cpp -Wall --std=c++17 -fsanitize=undefined
$ ./a.out
6 1 1 // 这行是输入的
a.cpp:14:9: runtime error: index -1 out of bounds for type 'int [2020]'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior a.cpp:14:9 in 
1 2 3 4 5 6
```

会发现这个代码访问了数组里下标为`-1`的地方。

上面的几段代码有什么共同点呢？为什么在平台上的执行结果就和本地不一样呢？为什么这几段代码表现出来好像“平台不好用了”呢？要回答这个问题，首先要理解“未定义行为”的概念。

## 什么是未定义行为

在我们学习编程的过程中，可能都知道一些行为是“非法”的，是“错误”的，比如：

1. 数组越界访问
2. 解引用空指针
3. 在对象的生命周期结束后访问对象
4. 有返回类型的函数没有从return结束

但是，我们也仅仅知道这些行为是“错误”的，并不会知道这些行为为什么错误，会造成哪些后果。这些行为在c++语言标准里有一个名字：未定义行为（undefined behavior）。如：

> 控制流出有返回值的函数（除了 main）的结尾而不经过 return 语句是未定义行为。

翻译成人话就是，除了main之外的函数，如果他有返回值但是不经过return语句就结尾了，行为未定义。

标准中也明确给出了未定义行为的解释：

> - *未定义行为（undefined behavior，UB）*——对程序的行为无任何限制。未定义行为的例子是数组边界外的内存访问，有符号整数溢出，空指针的解引用，在一个表达式中对同一标量[多于一次](https://zh.cppreference.com/w/cpp/language/eval_order)的中间无序列点 (C++11 前)无序 (C++11 起)的修改，通过[不同类型的指针](https://zh.cppreference.com/w/cpp/language/reinterpret_cast#.E7.B1.BB.E5.9E.8B.E5.88.AB.E5.90.8D.E5.8C.96)访问对象，等等。编译器不需要诊断未定义行为（尽管许多简单情形确实会得到诊断），而且所编译的程序不需要做任何有意义的事。

甚至：

> - [未定义行为能导致时间旅行（在所有事项中，时间旅行可是最惊人的）](http://blogs.msdn.com/b/oldnewthing/archive/2014/06/27/10537746.aspx)

翻译成人话就是，如果发生未定义行为，编译器可以把这段代码编译为任何内容，包括但不限于删除你的所有文件，帮你定一杯咖啡，或者时间旅行。**这些都是严格符合C++语言标准的。**同时，不同的编译器也会对未定义行为采取不同的策略，所以很可能未定义行为的代码在不同编译器上运行结果不同。这样定义”未定义行为“就使得编译器优化更好：很多情况下既然结果未定义，就可以没有结果，因此编译器可以去掉未定义行为发生的代码分支，把代码优化为行为确定的结果。

不理解上面那段话什么意思？没关系，我们回过来看之前的那段代码：

```cpp
template<typename T> bool ArrayList<T>::append(T const& value) {
  if(_size >= _capacity){ // border check;
    cout<<"SIZE "<<_size<<" CAP "<<_capacity;
    cout << " The List is overflow!" << endl;
    return false;
  }
  _elem[_size++] = value;
}
```

这段代码中实现了一个顺序表的append方法。看上去没什么问题：由于没有实现扩容算法，在末尾插入时首先要进行边界检查。如果边界检查通过，就把`value`放到`elem`里，并`size++`。但是，上面说了，这段代码的运行结果是`if`内条件永远成立，即使后面输出的时候`size`为`0`，`cap`为`2000`。为什么会这样呢？这是不是编译器的Bug？

其实并不是。可能你已经注意到了：这个函数最后缺少`return`。因此，`if`内条件不成立的行为未定义。编译器发现了这一点，认为程序员会保证每次调用的时候`if`内条件都成立（否则就会出现未定义行为），因此直接去掉了`if`，把代码编译成了大概这个样子：

```cpp
template<typename T> bool ArrayList<T>::append(T const& value) {
  cout<<"SIZE "<<_size<<" CAP "<<_capacity;
  cout << " The List is overflow!" << endl;
  return false;
}
```

重新回顾这个优化的过程中编译器的思路：

1. `if`内条件不成立的话，行为未定义。
2. 未定义行为不好，写代码的人会避免未定义行为。
3. 因此，写代码的人会保证每次调用的时候`if`内条件均成立。
4. 因此可以去掉`if`。

可以发现，整个优化过程中编译器严格的遵守了C++语言标准：

- 如果`if`内条件成立，这样的执行结果自然是正确的
- 如果`if`内条件不成立，那么行为未定义。既然行为未定义，那任何行为都是正确的，因此我执行`if`内条件成立的代码也是正确的行为。

因此，错的不是编译器，是~~整个世界~~你。编译器没那么多bug，当你以为编译器出了bug，绝大部分情况下是你写了bug。

~~当然，go编译器里还是不少bug的，这个之后再说~~

因此，到现在你应该知道了什么是未定义行为。**任何包括未定义行为的代码运行结果恰好符合你的预期都是巧合**。**任何时候不应该写有未定义行为的代码**。**未定义行为会导致代码在不同平台不同编译器上运行结果不一致**。

## 未定义行为的危害

未定义行为对于代码的危害上面已经说的差不多了。你可能会觉得：顶多代码运行结果是错的，又会怎么样呢？**NAIVE！**前面说道，当遇到未定义行为时：

> 编译器可以把这段代码编译为任何内容，包括但不限于删除你的所有文件，帮你定一杯咖啡，或者时间旅行。**这些都是严格符合C++语言标准的。**

不要以为“删除你的所有文件”是危言耸听：有人发现在clang编译器下你真的可能因为未定义行为而格式化你的硬盘：

![image-20210404121149762](/medias/image-20210404121149762.png)

这段代码首先定义了一个函数`f1`。在这个函数中，`i`会不断累加，直到溢出。**有符号数溢出是未定义行为**。于是，编译器没有给`f1`生成任何代码，只生成了一个label。

从右侧的汇编可以看到，`f1`label下面的代码就是`f2`函数，这个函数会格式化你的硬盘（在这个例子并没有真的格式化，注释掉了。）学过汇编的同学可能会发现，由于`f1`内没有任何代码，所以调用`f1`会执行你本来不想执行的`f2`函数。BOOM！你的硬盘被格式化了。

在[llvm的issue tracker中](https://bugs.llvm.org/show_bug.cgi?id=49599)有关这个 “bug”的讨论还在进行中。一部分人认为应该“修复”：

> This means UB is a potential safety/security problem, and we really should do something about it.

还有一部分人认为不应该：

> All sorts of UB manifests in lots of security issues, right? Buffer overruns and the like (I guess this is a buffer overrun, of sorts).

同时，有些人认为应该修复，理由是为了debug更方便。llvm的开发者回复：

> It's not generally that simple - deciding where/how to "recover" from UB would be pretty difficult.
>
> The Clang-advised way to deal with this would be to compile with -fsanitize=undefined
>
> https://godbolt.org/z/3aW69c
>
> example.cpp:4:29: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
> SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior example.cpp:4:29 in 

大意是已经有足够强大的未定义行为检测器了，为了方便debug来改这个UB的行为不值得。

总之，**不要写未定义行为**，以及**当你以为编译器错了，你错了。**~~我的OJ同理，因为就是帮你调用编译器来编译代码而已......~~



