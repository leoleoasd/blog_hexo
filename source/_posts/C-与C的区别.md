---
title: C++与C的区别
date: 2019-11-09 19:17:26
tags: 公众号同步推送
---
# CPP 与 C

### 区别1: 头文件不同

C++头文件没有.h

```cpp
C语言:
#include <stdio.h>

C++语言:
#include <cstdio>
#include <stack> // C++头文件
```

如果要在c++代码中引用c语言的库, 去掉.h, 并在最前面加上c.

```cpp
string.h >>> cstring
stdio.h  >>> cstdio
stdlib.h >>> cstdlib
...
```

### 区别2: 读入输出方式不同.

**c++语言中只要引用了库吗仍然可以用c语言的输入输出函数**.

```cpp
C++:
#include <iostream>
using namespace std;
int main(){
    int a;
    char b;
    char str[100] = {0};
    cin>>a>>b>>str;
    // 读入 cin 右箭头 变量名
    // 不需要&
    cout<<a<<b<<str<<endl;
    // 输出 cout 左箭头 变量名
    // endl和输出"\n"几乎等价, 但尽量用endl
}
```

### 区别3: 命名空间

死记硬背即可, c++代码需要在引用头文件后添加一行
```cpp
using namespace std;
```

### 区别4: 布尔类型
c++语言自带布尔类型. 存储的是真和假两种情况.
```cpp
bool a = false; //假, 与0等价
a = true; // 真, 与1等价
// 注意不要写为ture和flase
if(a){
    foo();
}else{
    bar();
}
```

~~c语言引用stdbool.h后也有, 但是课内不讲~~