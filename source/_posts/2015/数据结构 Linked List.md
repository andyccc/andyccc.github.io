---
title: 数据结构 Linked List
date: 2015-09-03 22:11:10
categories: 
-数据结构
tags:
- 数据结构
- Linked
- List
---


> Linked List 这里主要使用 Swift 实现链表及其一些基本特性. 12345678910111213141516171819202122232425262728293031323334353637......


[](#Linked-List "Linked List")Linked List
-----------------------------------------

这里主要使用 Swift 实现链表及其一些基本特性.

```
class LinkedListNode<T> {
    var value: T!
    var pre: LinkedListNode<T>?
    var next: LinkedListNode<T>?

    init(value: T) {
        self.value = value
    }

    func visit() {
        print(self.value)
    }

    func isEqual(_ node: LinkedListNode) -> Bool {
        return (value as! String) == (node.value as! String)
    }
}

class LinkedList<T> {
    typealias Node = LinkedListNode<T>

    var head: Node?

    var isEmpty: Bool {
        return head == nil
    }

    var first: Node? {
        return head
    }

    var last: Node? {
        guard var node = head else { return nil }

        while let next = node.next {
            node = next
        }
        return node
    }
}
```

### [](#遍历 "遍历")遍历

```
func enumerate(by closure: (_ node: Node) -> Void) {
    guard var node = head else { return }
    closure(node)

    while let next = node.next {
        node = next
        closure(node)
    }
}
```

### [](#操作 "操作")操作

#### [](#insert "insert")insert

```
func insert(_ node: Node, at index: Int) {
    if index < 0 || index > count { return }
    if count == 0 {
        head = node
        return
    }

    if index == 0 {
        node.next = head
        head?.pre = node
        head = node
    } else if index > 0 {
        if let pre = self.node(at: index - 1) {
            if let preNext = pre.next {
                node.next = preNext
                pre.next!.pre = node
            }
            pre.next = node
            node.pre = pre
        }
    }
}
```

#### [](#remove "remove")remove

```
func remove(_ node: Node) {
    let pre = node.pre
    let next = node.next

    if let pre = pre {
        pre.next = next
    } else {
        head = next
    }

    if let next = next {
        next.pre = pre
    }
}

func removeAll() {
    head = nil
}
```

#### [](#append-pop "append/pop")append/pop

```
func append(_ node: Node) -> Int {
    guard let last = last else {
        head = node
        return 0
    }
    last.next = node
    node.pre = last
    return count - 1
}

func pop() -> Node? {
    if count > 1 {
        let tmp = last!
        last!.pre!.next = nil
        return tmp
    } else if count == 1 {
        let tmp = head
        head = nil
        return tmp
    } else {
        return nil
    }
}
```

#### [](#contains "contains")contains

```
func contains(_ node: Node) -> Bool {
    guard var pivot = head else {
        return false
    }

    if pivot.isEqual(node) {
        return true
    }

    while let next = pivot.next {
        pivot = next
        if pivot.isEqual(node) {
            return true
        }
    }

    return false
}
```

### [](#反转 "反转")反转

```
func reverse() {
    var node = head
    while let curNode = node {
        node = curNode.next
        curNode.next = curNode.pre
        curNode.pre = node
        head = curNode
    }
}
```

### [](#是否有环 "是否有环")是否有环

利用快行指针来判断链表是否有环.

```
func isCircleExisting() -> Bool {
    var slow = head
    var fast = head

    while fast != nil && fast!.next != nil {
        slow = slow!.next
        fast = fast!.next!.next

        if let slow = slow, let fast = fast {
            if slow.isEqual(fast) {
                return true
            }
        }
    }
    return false
}
```

[](#示例代码 "示例代码")示例代码
--------------------

[swift_algorithm](https://github.com/andyccc/swift_algorithm)

[](#参考 "参考")参考
--------------

[swift-algorithm-club](https://github.com/raywenderlich/swift-algorithm-club)  
[Swift 算法实战之路：链表](https://www.jianshu.com/p/cf962aeff643)

坚持原创技术分享，您的支持将鼓励我继续创作！ So，来杯咖啡？
