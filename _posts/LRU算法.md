---
title: LRU算法
---

使用mac的时候，cmd+tab切换应用非常舒服。其中的算法就是LRU算法。LRU全称Least Recently Used，即最近最少使用算法。下面用js实现以下最简单的LRU算法。
<!--more-->

把我们的应用标记为A，B，C，D

我们依次打开应用，切换时应用的顺序为：D->C->B->A

当我们切换到应用B后，应用的排序就变成了：C->D->B->A

当前应用的下一个应用就是上一次使用的应用。这就使的在两个应用间来回切换时就变得非常的便利。

以下用链表来实现最简单的LRU算法：

```javascript
/// Definition for a binary tree node.
function LinkNode(val) {
  this.val = val;
  this.prev = this.next = null;
}

let head = new LinkNode()

function addNode(val) {
  let curNode = new LinkNode(val)
  /// 新节点插入到最前面
  if (head.next) head.next.prev = curNode
  curNode.next = head.next
  curNode.prev = head
  head.next = curNode
  return curNode
}

function removeNode(node) {
  /// 当前节点的前后两个节点连接
  node.prev.next = node.next
  node.next.prev = node.prev
}

function bringToFront(node) {
  if (head.next === node) return 
  /// 当前节点的前后两个节点连接
  node.prev.next = node.next
  node.next.prev = node.prev
  /// 当前节点插到最前面
  if (head.next) head.next.prev = node
  node.next = head.next
  head.next = node
  node.prev = head
}

function log() {
  let result = ''
  let node = head.next
  while(node) {
    result += node.val
    node = node.next
  }
  console.log(result)
}

let nodeA = addNode('a')
let nodeB = addNode('b')
let nodeC = addNode('c')

log()
/// expected cba

bringToFront(nodeB)

log()
/// expected bca
```

LRU的算法实现原理其实并不复杂，但简单的原理在生活中有着很好的表现。可能也是算法之美——化繁为简。