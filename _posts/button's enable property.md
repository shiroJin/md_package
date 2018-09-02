---
title: button's enable property
---

​	首先看一段代码：

```objective-c
	button.enable = NO;
		[self mock] `// here do something time-consuming synchronous`
	button.enable = YES;
```

	很明显，这段代码的意图在与防抖。在耗时任务前关闭按钮的可点击状态，然后进行耗时任务，任务结束之后再打开。但是测试结果发现，enable属性并没有生效。
	先谈谈runloop，runloop是iOS应用可以响应的根本。线程执行完任务之后便会结束线程的周期。但，主线程不会，主线一直在工作。回归主题，讲讲页面是如何显示的。页面元素的信息并不是每次在线程执行完修改页面的代码后立刻生效。当runloop一圈走完后，runloop会将图层树上的所有修改打包提交到渲染中心，最后显示的屏幕上。事件的响应也如此，在runloop没有提交enable属性的修改前，按钮一直是可点击状态。
	对于以上防抖的处理，则需要先将enable属性的修改提交上去。当然将耗时任务放到异步线程中处理是个不错的选择。
```objective-c
	button.enable = NO;
	    dispatch_async(dispatch_get_global_queue(0,0), ^{
        // do timing-consuming work here
        dispatch_async(dispatch_main(), ^{
        	button.enable = YES;
    	});
    });
```

​	或者，有另一种方案，我们需要确保enable属性先提交，然后下一次loop中处理耗时任务。我们可以延时调用来处理任务。

```objective-c
	button.enable = NO;
	dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 0.25f, dispatch_get_main_queue(), ^{
		// do timing-consuming work here
		button.enable = YES;
	};
```

