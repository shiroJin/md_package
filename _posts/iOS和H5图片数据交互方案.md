---
title: iOS和H5图片数据交互方案
---

## 需求场景

H5页面使用客户端的相册图片。H5调用客户端的相册功能选取图片，图片资源在客户端本地，因此需要客户端将选取后的图片数据返回给H5。

## 方案设计

#### 一、直接通过桥将数据传给H5应用

将获取的UIImage对象转化为NSData对象，通过桥，直接将NSData对象传给H5应用。

#### 二、拦截H5的网络请求，将图片数据通过网络请求的方式返回给H5应用

客户端图片选取之后，生成一个图片地址，地址中包含特定的图片协议和这个图片的唯一标识符。图片地址返回给H5应用，H5应用取得图片地址后，将图片地址视为一个网络图片地址，通过request请求图片数据。客户端通过NSURLProtocol拦截WebView发起的包含特定图片协议的网络请求，并根据请求的地址匹配本地的图片资源，生成response返回给H5应用。客户端在请求过程中扮演一个本地服务器的角色。
<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/iOS%E5%92%8CH5%E5%9B%BE%E7%89%87%E4%BA%A4%E4%BA%92%E6%96%B9%E6%A1%88_1.png">

## 缓存设计

相册选择完图片后，已经获取到了图片数据，对应的图片数据进行缓存，提高读取效率。获取到的UIImage对象转化为NSData数据存储在缓存池中，缓存池的结构为简单的hash表。

### 拦截器实现

iOS中系统提供了请求的拦截器——NSURLProtocol。子类化NSURLProtocol类，并注册这个urlProtocol即可拦截到当前进程的网络请求。但在 WKWebView 中的请求却完全不遵从这一规则，WKWebView有自己独立的进程，通过IPC进行进程间通信。想要拦截WKWebView的请求则需要通过c类的registerSchemeForCustomProtocol:方法进行注册urlProtocol(WKBrowsingContextController是私有类，只能通过runtime的方式生成获取这个类，且iOS 8.4 以后才被引入，调用私有类和私有方法存在被拒的隐患)。

```objc
// 注册protocol scheme为自定义的protocol。
Class cls = [[[WKWebView new] valueForKey:@"browsingContextController"] class];
SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
if ([(id)cls respondsToSelector:sel]) {
	[(id)cls performSelector:sel withObject:scheme];
}
```

### 降级方案

对于iOS 8.4 以下的版本，无法获取到WKBrowsingContextController类，也就导致无法进行网络拦截。在整个图片选择逻辑中，APP通过桥传输的都是图片的地址。H5应用都是直接加载图片地址的数据。因此，对于iOS 8.4以下的版本，APP端不作本地服务器，选择完图片后自行传到远程服务器，将正常的网络图片地址传给H5应用。
