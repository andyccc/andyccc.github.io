---
title: 数据结构 Stack 与 Queue
date: 2015-08-11 21:20:10
categories: 
-数据结构
tags:
- 数据结构
- Stack
- Queue
---


> StackStack 是一个后进先出的数据结构, LIFO(Last In First Out). 其内部可以使用 Array 来实现. push/pop 操作分别从 Array 的尾部添加 / 删除元素, 时间复杂度均......


[](#Stack "Stack")Stack
-----------------------

Stack 是一个后进先出的数据结构, LIFO(Last In First Out). 其内部可以使用 Array 来实现.  
push/pop 操作分别从 Array 的尾部添加 / 删除元素, 时间复杂度均为 O(1).  
注意: 不能从 Array 的首部进行操作, 因为首部插入数据需要移动整个数组中的元素, 时间复杂度比较高.

```
public struct Stack<T> {
    fileprivate var array = [T]()

    public var isEmpty: Bool {
        return array.isEmpty
    }

    public var count: Int {
        return array.count
    }

    public mutating func push(_ newElement: T) {
        array.append(newElement)
    }

    public mutating func pop() -> T {
        return array.remove(at: array.count - 1)
    }

    public var top: T? {
        return array.last
    }
}
```

[](#Queue "Queue")Queue
-----------------------

Queue 是先进先出的结构, FIFO(First In First Out), 内部也可采用 Array 来实现.  
enqueue 将元素加入队列, 可直接将元素放在 Array 的尾部, 时间复杂度为 O(1).  
dequeue 将首元素移出队列, 将 Array 的首元素移出, 其他元素的位置均需调整 (向前 shift), 因此 dequeue 操作的时间复杂度为 O(n).

```
public struct Queue<T> {
    fileprivate var array = [T]()

    public var isEmpty: Bool {
        return array.isEmpty
    }

    public var count: Int {
        return array.count
    }

    public mutating func enqueue(_ newElement: T) {
        array.append(newElement)
    }

    public mutating func dequeue() -> T? {
        if isEmpty {
            return nil
        } else {
            return array.remove(at: 0)
        }
    }

    public var front: T? {
        return array.first
    }
}
```

### [](#dequeue操作的优化 "dequeue操作的优化")dequeue 操作的优化

Queue 的 enqueue 操作非常高效, 但 dequeue 操作却有很多优化的空间.  
dequeue 之后, 并不将内部 Array 的其他元素依次向前 shift, 而是将 dequeue 之后的空位留着, 定期将所有空位移除即可. (即 Array 中可以存储空值).  
这样, 尽管定期移除所有空位的操作的时间复杂度为 O(n), 但 dequeue 操作的平均时间复杂度却降为了 O(1).  
这种优化方法需要引入一个变量 head, 用于存储当前的实际首元素的索引位置.

```
public struct Queue_Optimized<T> {
    fileprivate var array = [T?]()
    fileprivate var head = 0 // 真正的首元素所在的index

    public var isEmpty: Bool {
        return count == 0
    }

    public var count: Int {
        if head < array.count {
            return array.count - head
        } else {
            return 0
        }
    }

    public mutating func enqueue(_ newElement: T) {
        array.append(newElement)
    }

    public mutating func dequeue() -> T? {
        guard count > 0, let headElement = array[head] else { return nil }

        array[head] = nil
        head = head + 1

        // remove nil elements
        let p = Double(head) / Double(array.count)
        if array.count > 100 && p > 0.25 {
            // removeFirst(_:) Removes the specified number of elements from the beginning of the collection.
            array.removeFirst(head)
            head = 0
        }

        return headElement
    }

    public var front: T? {
        return array[head]
    }
}
```

[](#示例代码 "示例代码")示例代码
--------------------

[swift_algorithm](https://github.com/andyccc/swift_algorithm)

[](#参考 "参考")参考
--------------

[swift-algorithm-club](https://github.com/raywenderlich/swift-algorithm-club)

坚持原创技术分享，您的支持将鼓励我继续创作！ So，来杯咖啡？
