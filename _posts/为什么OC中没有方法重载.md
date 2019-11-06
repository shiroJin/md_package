---
title: 为什么OC中没有方法重载
---

#### 前言：什么是函数重载？

在C++和java语言中都有函数重载是指在同一作用域内，可以有一组具有相同函数名，不同参数列表的函数，这组函数被称为重载函数。重载函数通常用来命名一组功能相似的函数，这样做减少了函数名的数量，对于程序的可读性有很大的好处。例如：

```C++
void foo(int i) {
  cout<<"print a integer :"<<i<<endl;
}

void foo(string str) {
  cout<<"print a string :"<<str<<endl;
}

int main() {
  foo(12);
  foo("hello world!");
  return 0;
}
```

main函数中foo(12)调用的是foo(int)函数，foo("hello world")调用foo(string str)函数。

#### C++，Java中是如何实现方法重载

C++和Java允许重载方法，这些方法在代码中都有相同的名字，参数列表不同。如果仅仅将方法名最为符号，那链接器定然不能识别。之所以能使用重载是因为他们的编译器将方法名和参数列表组合编码作为这个方法的符号。这种编码过程称为重整(mangling)。重整后的方法因为参数列表不同，编码后得到的符号也不相同。例如foo(int)编码为\_\_Z3fooi, foo(string)编码为\_\_Z3fooSs。

#### OC中方法有重载吗？

试想一下在一个类中添加如下两个方法。

```objc
- (void)foo:(int)i;
- (void)foo:(NSString *)str;
```

毫无疑问会收到报错。错误内容为<font color=red>Duplicate declaration of method</font>，方法名重复了。显然OC不支持这样的写法。

#### 理解OC中的方法调用

```objc
[self foo]
```

通过clang转为c++代码为

```c++
((void (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("foo"));
```

foo方法的调用，实际是调用了objc_msgSend这个函数，self，sel作为函数的参数。

再来看下上述两个方法。两者的sel都是<em>foo:</em>。对于一个类，其内部有一个方法的列表。msgSend函数的实现为根据sel从方法列表中获取imp。因此上述两个方法的SEL存在重复，msgSend函数不能区分，找到对应的imp。因此，在OC语言中不存在上述的方法重载。

