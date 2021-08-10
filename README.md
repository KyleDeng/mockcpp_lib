# mockcpp使用

[TOC]

---

参考资料：

[mockcpp环境搭建](https://blog.csdn.net/ccy00808/article/details/107051259)

[mockcpp使用方法指导](https://blog.csdn.net/xueyong4712816/article/details/34086649)

[MockCpp手册](https://blog.csdn.net/Tony_Wong/article/details/38752355)

---

## 0. 安装
本机环境： `Ubuntu 20.04`

mockcpp下载链接：`https://code.google.com/archive/p/mockcpp/downloads`

修改顶层的`CMakeLists.txt`:
```cmake
PROJECT(mockcpp)
add_definitions(-std=c++11)  # 加上这句话
```

解压以后，通过`cmake`编译，
```cmake
cmake -B build
cmake --build build
cmake --install build --prefix install
```

然后在`install`目录中可以看到产物，放到项目中或者
放到`/usr/lib`和`/usr/include`中，成为标准库。

另外`mockcpp`使用需要依赖`boost`库，安装：`sudo apt-get install libboost-dev `。

### 0.1 可能遇到的问题
1. python3不兼容
```log
Traceback (most recent call last):
  File "/home/huatuo/tmp/mockcpp/src/generate_vtbl_related_files.py", line 5, in <module>
    from get_long_opt import *
  File "/home/huatuo/tmp/mockcpp/src/get_long_opt.py", line 29
    print sys.argv[0], getUsageString(longOpts)
```
**解决：**
在`/usr/bin`中使用`ln -s`临时修改`python`指向`python2`，**记得编译成功后要修改回来**。

2. `static_assert`重定义
```log
In file included from /home/huatuo/tmp/mockcpp/include/mockcpp/AfterMatcher.h:24,
                 from /home/huatuo/tmp/mockcpp/src/AfterMatcher.cpp:19:
/home/huatuo/tmp/mockcpp/include/mockcpp/mockcpp.h:59:8: error: expected identifier before ‘static_assert’
   59 | struct static_assert
      |        ^~~~~~~~~~~~~
/home/huatuo/tmp/mockcpp/include/mockcpp/mockcpp.h:59:8: error: expected unqualified-id before ‘static_assert’
```
**解决：**`vim ./include/mockcpp/mockcpp.h +59`把以下内容删除
```c++
template <bool condition>
struct static_assert
{
    typedef int static_assert_failure[condition ? 1 : -1];
};
```


## 1. 使用
概览：
```c++
#include "mockcpp/mockcpp.hpp"

TEST(mockcpp simple sample)
{
    MOCKER(function) / MOCK_METHOD(mocker, method)
        .stubs() / defaults() / expects(once())
        [.before("some-mocker-id")]
        [.with(eq(3))]
        [.after("some-mocker-id")]
        .will(returnValue(1)) / .will(repeat(1, 20))
        [.then(returnValue(2))]
        [.id("some-mocker-id")]
}
```

详细的使用情况，带有完整的扩展关键字：
```c++
TEST(mockcpp detail sample)
{
    MOCKER(function) / MOCK_METHOD(mocker, method)
        .stubs() / defaults() / expects(never() | once() | exactly(3) | atLeast(3) | atMost(3) )
        [.before("some-mocker-id")]
        [.with( any() | eq(3) | neq(3) | gt(3) | lt(3) | spy(var_out) | check(check_func)
                | outBound(var_out) | outBoundP(var_out_addr, var_size) | mirror(var_in_addr, var_size)
                | smirror(string) | contains(string) | startWith(string) | endWith(string) )]
        [.after("some-mocker-id")]
        .will( returnValue(1) | repeat(1, 20) | returnObjectList(r1, r2)
                | invoke(func_stub) | ignoreReturnValue()
                | increase(from, to) | increase(from) | throws(exception) | die(3))
        [.then(returnValue(2))]
        [.id("some-mocker-id")]
}
```

### 1.1 `MOCKER / MOCK_METHOD`
> `MOCKER`： C函数或者类的静态成员方法；
>
> `MOCK_METHOD`：类的非静态成员的方法需要先用`Mock<MyClass>mocker`声明一个mock对象，再用`MOCK_METHOD(mocker, method)`来mock指定方法。

### 1.2 `stubs / defaults / expects`
在使用`MOCKER/MOCK_METHOD`之后，必选从`stubs()/defaults()/expects()`中选择其一。
> `stubs()`：指定函数的行为，不校验函数的执行次数；
>
> `expects()`：不仅指定函数的行为，还要校验函数的执行次数；
>
> `defaults()`：定义一个mock的默认行为，优先级最低，一旦后面有`stubs`或者`expects`就会重新指定行为，一般用在`setUp`中。

`expects()`中的参数介绍：
>`never()`    : 从不被调用
>
>`once()`     : 只调用一次
>
>`exactly(3)` : 被调用三次
>
>`atLeast(3)` : 最少被调用三次
>
>`atMost(3)`  : 最多被调用三次
 
### 1.3 `will / then`
用来指定函数的行为。

带有返回值的函数，MOCKER后面必须有will，否则mockcpp认为无返回值，校验时会发现返回类型不匹配。

```c++
.will(retunValue(1))  //函数第一次执行时返回1
.then(returnValue(2)) //函数第二次执行时返回2
.then(returnValue(3)) //函数第三次及以后执行时返回3
```

```c++
repeat(1, 20)            // 调用20次都是返回1
returnObjectList(r1, r2) // 第一次返回r1，第二次返回r2
invoke(func_stub)        // 用func_stub函数代替原函数
ignoreReturnValue()      // 忽略函数的返回值
increase(from, to)       // 表示依次返回from到to的对象，任何重载了 +  + 运算符的对象都可以用
throws(exception)        // 抛出exception异常
die(3)                   // 退出程序并返回指定的值
```

### 1.4 `with`
当需要函数根据不同的参数而做出不同的行为时，可以使用`with`。
```c++
MOCKER(add)
    .stubs()
    .with(eq(1), lt(10))  //当第一个参数等于1，第二个参数小于10时
    .will(returnValue(0)) //函数返回0

MOCKER(add)
    .stubs()
    .with(eq(1), gt(10))  //当第一个参数等于1，第二个参数大于10时
    .will(returnValue(99)) //函数返回99
```

```c++
any()                             //任何参数
eq(3)                             //等于
neq(3)                            //不等于
gt(3)                             //大于
lt(3)                             //小于
spy(var_out)                      //监视函数的入参并保存在var_out中，供用例其他地方使用
check(check_func)                 //用函数定制入参的检查规则
outBound(var_out)                 //设置函数出参的值
outBoundP(var_out_addr, var_size) //同上，但作用于指针、数组一类
mirror(var_in_addr, var_size)     //针对指针类型参数，内容匹配
smirror(string)                   //参数与字符串string内容匹配
contains(string)                  //参数中包含string
startWith(string)                 //参数以string开头
endWith(string) )                 //参数以string结尾
```

指定出参的例子：
```c++
void function(int a, int *val)
{
}

TEST(Suit, Case)
{
    int expect = 10;
    int expect2 = 99;

    MOCKER(function)
      .stubs()
      .with(eq(1), outBoundP(&expect, sizeof(expect)));  //当参数a=1时，设置出参为expect

    MOCKER(function)
      .stubs()
      .with(eq(9), outBoundP(&expect, sizeof(expect)));  //当参数a=9时，设置出参为expect2
}

```

### 1.5 `id / before / after`
用`id()`给mock规范指定一个名字，然后使用`before()`和`after()`来指定多个mock的调用顺序。

**注意：**before在with前，after在with后，id在整个mock规范的最后。

```c++
MOCKER(add)
  .stubs()
  .with(eq(1), eq(2))
  .will(returnValue(30))
  .id("first");  //指定这个mock的名字是first

MOCKER(add)
  .stubs()
  .with(eq(3), eq(4))
  .after("first")  //只能在first以后被触发，否则用例失败
  .will(returnValue(700));
```

## 2. `verify() 和 reset()`
### 2.1 `reset()`
当我们使用了`MOCKER/MOCK_METHOD`后，如果需要取消这些打桩的内容，我们可以调用：
```c++
GlobalMockObject::reset();
```

例如：
```c++
TEST(Suit, Case)
{
    MOCKER(function)
      .stubs()
      .with(eq(1))
      .will(returnValue(1));

    int ans = hello_world(1); //hello_world中调用了function

    ASSERT_EQ(ans, 1);

    GlobalMockObject::reset(); //取消对function的mock
}

TEST(Suit, Case_2)
{
    //如果没有使用过reset，function依然使用的mock过的

    int ans = hello_world(1); //hello_world中调用了function

    ASSERT_EQ(ans, 1);
}
```

### 2.2 `verify()`
对于下面的测试用例，我们指定`function`被调用2次，但是实际上我们只使用了1次。

在这种情况下**并不会报错**。

我们需要在用例最后加上**`GlobalMockObject::verify();`**，才能检查mock期望是否正常。

```c++
TEST(Suit, Case)
{
    MOCKER(function)
      .expects(exactly(2))
      .will(returnValue(1))
      .then(returnValue(100));

    int ans = hello_world(1); //hello_world中调用一次function

    ASSERT_EQ(ans, 1);
}
```

**注意：**`verify()`中会默认调用`reset()`。

---

