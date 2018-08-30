---
title: viewController未出现前强制加载View
---
## 1.viewcontroller的生命周期

loadView -> viewDidLoaded -> viewWillAppear
有没有覆写loadView方法；
xib加载

## 2.实例
viewController还没加载出来，对视图上的UI元素进行了操作，这时候因为loadView, viewDidLoaded没有被调用，UI控件还没被初始化，
此时外部的方法调用对UI元素并没有任何作用。所以需要显式的调用loadView方法，loadView方法一般viewController自己隐式的调用，
而且只调用一次。所以这里可以通过访问viewController.view来强制viewController去调用loadView，viewDidLoaded方法。