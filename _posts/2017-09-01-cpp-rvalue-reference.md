---
layout: post
title: 我对C++11右值引用的理解
category: another
---

# lvalue reference和rvalue reference

首先，什么是lvalue和rvalue？

在C中，lvalue是=左边,rvalue是=右边。在C++中，略有不同，不过可以这样简单理解：如果一个object是一个lvalue，我们用它本身(内存地址)，
如果一个object是一个rvalue，我们用它的值（内存中的内容）。

那么，什么是lvalue reference和rvalue reference?

C++中正常的引用，就是lvalue reference，它是对另一个lvalue的引用，如`int b = 1; int& a = b;`。而rvalue reference是对一个rvalue的引用，例如`int&& a = 1;`。
C++之所以出现rvalue reference，主要是为了效率。因为rvalue是一个临时的东西，所以可以保证rvalue reference指向的东西只有它自己可以用，别人动不到这块内存，
因此这这块内存可以安全地任意折腾。

需要注意的是，在`int&& a = 1;`中，a是对1的rvalue reference，但是a本身是一个lvalue，毕竟a有name，有地址。

# std::forward
std::forward用于完美转发（其实就是个强制类型转化啦）。考虑如下函数：
```
template<typename T>
void Foo(T&& t) {
  SomeFunc(std::forward<T>(t));
}
```
std::forward的作用则是t传给Foo是什么类型，就将什么类型原封不动地传给SomeFunc。

它是怎么实现的呢？直接给出std的source code:
```
// Overload 1
template<typename _Tp>
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type& __t) noexcept
    { return static_cast<_Tp&&>(__t); }

// Overload 2
template<typename _Tp>
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
    {
      static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
		    " substituting _Tp is an lvalue reference type");
      return static_cast<_Tp&&>(__t);
    }    
```

当调用`Foo(1);`时，实参是一个rvalue。由于T的类型是`int`,t是一个lvalue，则调用std::forward<int>(int& t)版本（overload 1），该重载返回int&&，因此传给SomeFunc的
实参类型保持一致。

当调用`int a = 1; Foo(a);`时，实参是一个lvalue。由于T的类型是`int&`，t是一个lvalue，则调用std::forward<int&>(int& t)版本(overload 1)，该重载返回int&&& -> int&，因此传给SomeFunc的
实参类型保持一致。

什么时候会调用Overload 2呢？嘿嘿，试试std::forward<T>(std::forward<T>(t))并想想为什么吧。

# std::move
std::move用于将任意类型转换为rvalue reference类型（也是个强制类型转换啦）。直接看实现：
```
template<typename _Tp>
  constexpr typename std::remove_reference<_Tp>::type&&
  move(_Tp&& __t) noexcept
  { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
```
可以看到，无论传给std::move是lvalue还是rvalue，最终都是得到rvalue reference.

{% include references.md %}
