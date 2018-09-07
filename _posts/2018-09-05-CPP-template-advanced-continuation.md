---
title: CPP-template-advanced-continuation
key: 201809050
tags: CPP template
---

# CPP-template-advanced-continuation

- T(args...) 构造 => 匿名对象是(非const)右值, 类似函数**对象返回** => 难以体现右值性(与对象相关的右值) 
- 什么是右值 => `exp(除了[], *解引用)`值, 函数返回值, 匿名构造, 运算符相关
- 什么是左值 => 类型声明, 实例化对象
- 右值/左值差别 => 语义(semantic)相关, 运算符(operator)相关, 小命长短相关
- 注意const右值 当且仅当对于对象才有意义
- const右值引用也存在, 可以绑定const/非const右值
>只是为了语法的完整性而存在, 无法触发真正的移动语义 => 因为无法进行steal operation => 进而没B用
>如move constructor, move assignment, std::move()返回无意义啊
>将只会触发copy assignment, 即 const T& 范式
>

<!--more-->

#### `std::move()`的左值实参与右值实参详解

```CPP
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}

string s1 = "hi!", s2;
s2 = std::move(string("bye!")); //1 构造返回右值 => string move assignment
s2 = std::move(s1)              //2 string move assignment
```

- 对于#1
>T = string
>函数模板实例 string&& move(string&& t); => 右值引用绑定右值
>trival cast
>

- 对于#2
>T = string&
>函数模板实例 string&& move(string& t); => 左值引用绑定左值
>static_cast<string&&>(t) 是合法的
>即使用C++强制类型转换允许将左值引用/绑定后的左值截断(clopper/expire lvalue)为右值
>这意味着左值的资源管理权限已经交出, 小命玩儿完了! 移后源必会析构, 值不确定
>特例 cast
>

- 显然当实参有const限定时, std::move()的const右值引用返回任何意义啊, 与之前讨论相同

- `注意/结论: 放心使用std::move(非const l/r value)即可, 放心移动语义, 放心截断左值, 此处的特例自己少用啊! 当然实参const限定时DSB啊!`

#### 拷贝控制成员的编译器自动生成(synthesized)简述
- trival
   - 构造/赋值操作是`memberwise`进行的, 包括(非静态)成员和直接/间接基类
   - 不能`构造/赋值操作`的成员(刺头成员) 或 直接间接虚基类引发定义删除
   - 程序员自定义`拷贝控制成员`引发删除, 但是可以**强制**
   - 不能析构, 导致移动语义定义`move constructor/assignment`删除
   - `=default`具有**显式**和**强制**双重意义
- 没有任何构造时 => 生成无参构造, 定义是trival进行的, 否则声明为`=delete`
- `copy/move assignment`自动生成trival, const成员引发删除, 智只能自定义
- `copy/move constructor`自动生成trival
- 有`move constructor/assignment`声明时, `copy constructor/assignment`定义删除 => 原因是为了移动语义正确执行啊!

#### 转发或参数传递
>解决的问题是, 函数模板将参数再次传递时
>左值, 右值对象属性难以保证
>const限定符难以保证
>

- 先再次理解右值引用
   - 绑定右值后
   - 对外是移后源一定会析构, 值不确定
   - 对内, alias是**自由修改的左值**, 是左值, 事实正是如此
- 同理const右值引用
   - 不光无法实现对外的移后源保证
   - 而且alias是const左值 => read-only, 事实真是如此
   - 结果还是没B用
- 对应的左值引用, const左值引用trival => 很自然 => 即左值, const左值

```CPP
template<typename F, typename T1, typename T2>
void flip1(F f, T1 t1, T2 t2)
{
    f(t2, t1);
}

template<typename F, typename T1, typename T2>
void flip2(F f, T1&& t1, T2&& t2)
{
    f(t2, t1);
}

template<typename F, typename T1, typename T2>
void flip3(F f, T1&& t1, T2&& t2)
{
    f(std::forward<T1>(t2), std::forward<T2>(t1));
}

void f(int v1, int& v2);
int i;
flip1(f, i, 0);
/*
实例是 flip1(void(*)(int, int&), int, int);
i永远不会变
*/

flip2(f, i, 0);
/*
实例是 flip2(void(*)(int, int&), int&, int&&);
i会变, 显然, 而且会保持const左值/右值属性
i的变化过程是(左值 => 绑定到左值引用 => alias左值 => f调用)
但是0的变化过程是(右值 => 绑定到右值引用 => alias左值 => f调用)
问题是: 无法一直保持右值引用的特性
*/
void g(int&& v1, int& v2);
flip2(g, i, 0); // Error! 右值引用无法绑定左值lvalue

flip3(g, i, 0); // so good!
```

- `std::forward<T>()`常用来转发右值引用, 且显示模板参数为函数模板模板参数T, 与右值引用函数模板模板参数推导`T&&`配合使用 => 完美转发

#### 函数模板重载
- 函数模板之间, 函数之间, 函数与函数模板之间构成重载关系的条件相同
>即: 不看返回值, 只看参数类型, 个数, const, throw()声明符是否相同
>注意: 函数模板之间的重载可以使用相同的模板前缀, 也可使用不同的模板前缀
>使用相同的模板前缀时, 一般参数个数不同, 否则为函数模板特化(specialized)
>

- 重载决议匹配原则
   - 匹配的更好, 少执行转换(允许的转换中)
   - 更特例, 如`const T&` 与 `T *`面对`string *`或`const string *`
      - 面对前者, `T *`是更好的选择, 不需要const转换即可
      - 面对后者, `T *`是更好的选择, 更特化, 与模板特化同义
   - 多个候选版本中, 优先非模板版本
   - 详见`C++ Primer::P619`

```CPP
// 二者为函数模板之间的重载, 都可以匹配所有类型
// 但是
// 左值, const左值, const右值
template <typename T> void f(const T&); 
// 右值
template <typename T> void f(T&&);
```

#### (函数)模板参数变长 <=> 可变参数函数模板
> 函数参数变长
> 使用的简单方法之一是: 初始化列表, 即`Aggragate类型`的花括号
> 实现为`initilizer_list<T>`模板的元素 + 继承
> 此外`<cstdarg>::var_arg` 参数变长必有一个非变长参数, 即`...`之前有还有参数
> 

- 模板使用递归, 递归的base case终止条件是一个模板参数不变长的, 其余使用参数包`Sizeof(Args)`作为递归阶段
- 模板前缀一般不同, 多个参数包啊, 参数一般不同, 多个参数包啊
- 本质是函数模板之间的重载, 终止条件 => 匹配最好的版本, 更特化的版本
- `注意: 递归终止条件一定声明在前`, 否则无限递归啊, 参数包为0 => GG

```CPP
template <typename T>
ostream& print(ostream& out, const T& t)
{
    out << t;
}

// 在前缀和函数形参中的 ... 前后允许空白符
template <typename T, typename... Args>
ostream& print(ostream& out, const T& t, const Args&... rest)
{
    out << t << ", ";
    return print(out, rest...);
}
```

- 语法`...`的语义为`包扩展`, 语法格式为`参数包模式 ...`
- 意义为: `Python/JS::map()`或`std::transform()`,  使用返回的线性表作为又一个参数包(即声明的rest又是一个参数包)
- 如`类型参数包 -> 名字对象参数包`
- 参数包模式可以很复杂
   - 对与第一次扩展 => 是类型上的声明 => 可以CV限定, 引用
   - 之后的扩展 => 是name/标识符/对象/变量 => 可以是任何表达式

#### 成员对象/变量指针, 成员函数指针简述




