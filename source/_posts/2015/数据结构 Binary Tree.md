---
title: 数据结构 Binary Tree
date: 2015-08-22 21:19:10
categories: 
-数据结构
tags:
- 数据结构
- Binary
- Tree
---

> 二叉树没什么可介绍的, 这里主要用 Swift 来实现二叉树的一系列相关函数. 实现这些函数的关键在于 遍历与递归的使用. Binary Tree 二叉树的基本结构如下: 1234567891011121314......


二叉树没什么可介绍的, 这里主要用 Swift 来实现二叉树的一系列相关函数.  
实现这些函数的关键在于 **_遍历与递归的使用_**.

[](#Binary-Tree "Binary Tree")Binary Tree
-----------------------------------------

二叉树的基本结构如下:

```
class Node {
    var value: String!
    var left: Node?
    var right: Node?

    convenience init(value: String) {
        self.init()
        self.value = value
    }

    func visit() {
        print(value)
    }

    func isNodeEqualTo(node: Node) -> Bool {
        return value == node.value
    }

}
```

以下的计算属性及方法均添加在 Node 的 extension 中.

[](#一些简单属性 "一些简单属性")一些简单属性
--------------------------

### [](#最大深度 "最大深度")最大深度

二叉树的最大深度为 [log(N+1), N].

```
var maxDepth: Int {
    var number = 0
    if let left = left {
        number = left.maxDepth
    }
    if let right = right {
        number = max(number, right.maxDepth)
    }
    number = number + 1
    return number
}
```

### [](#所有节点 "所有节点")所有节点

二叉树的所有节点个数为 N.

```
var allNodesNumber: Int {
    var number = 0
    if let left = left {
        number = number + left.allNodesNumber
    }
    if let right = right {
        number = number + right.allNodesNumber
    }
    number = number + 1
    return number
}
```

所有节点:

```
var allNodes: [Node]! {
    var leftNodes = [Node]()
    var rightNodes = [Node]()
    if let left = left {
        leftNodes = left.allNodes
    }
    if let right = right {
        rightNodes = right.allNodes
    }
    return leftNodes + [self] + rightNodes
}
```

### [](#所有叶子节点 "所有叶子节点")所有叶子节点

叶子节点的 left 与 right 均为空.

二叉树的所有叶子节点的个数为 [1, log(N+1)].

```
var allLeafNodesNumber: Int {
    if left == nil && right == nil {
        return 1
    }

    var number = 0
    if let left = left {
        number = number + left.allLeafNodesNumber
    }
    if let right = right {
        number = number + right.allLeafNodesNumber
    }
    return number
}
```

所有叶子节点:

```
var allLeafNodes: [Node]! {
    if left == nil && right == nil {
        return [self]
    }

    var leafNodes = [Node]()
    if let left = left {
        leafNodes = leafNodes + left.allLeafNodes
    }
    if let right = right {
        leafNodes = leafNodes + right.allLeafNodes
    }
    return leafNodes
}
```

[](#第level层的节点 "第level层的节点")第 level 层的节点
----------------------------------------

二叉树的第 level 层的所有节点个数:

```
func nodesNumberAtLevel(level: Int) -> Int {
    if level < 1 {
        return 0
    }
    if level == 1 {
        return level
    }
    var number = 0
    if let left = left {
        number = number + left.nodesNumberAtLevel(level: level - 1)
    }
    if let right = right {
        number = number + right.nodesNumberAtLevel(level: level - 1)
    }
    return number
}
```

第 level 层的所有节点:

```
func nodesAtLevel(level: Int) -> [Node] {
    if level < 1 {
        return [Node]()
    }
    if level == 1 {
        return [self]
    }

    var leftNodes = [Node]()
    var rightNodes = [Node]()
    if let left = left {
        leftNodes = left.nodesAtLevel(level: level - 1)
    }
    if let right = right {
        rightNodes = right.nodesAtLevel(level: level - 1)
    }
    return leftNodes + rightNodes
}
```

访问第 level 层的所有元素:

```
func visitNodesAtLevel(level: Int) {
    for node in self.nodesAtLevel(level: level) {
        node.visit()
    }
}
```

[](#二叉查找树 "二叉查找树")二叉查找树
-----------------------

二叉查找树的特点：左结点的值 < 根节点的值 < 右节点的值.

```
var isBinarySearchTree: Bool {
    return p_isValueIn(min: nil, max: nil)
}

private func p_isValueIn(min: String?, max: String?) -> Bool {
    // 所有左子节点都必须小于根节点
    if min != nil && value <= min! {
        return false
    }
    // 所有右子节点都必须大于根节点
    if max != nil && value >= max! {
        return false
    }

    var ret = true
    if let left = left {
        ret = ret && left.p_isValueIn(min: min, max: value)
    }
    if let right = right {
        ret = ret && right.p_isValueIn(min: value, max: max)
    }
    return ret
}
```

[](#二叉树反转 "二叉树反转")二叉树反转
-----------------------

反转的要点在于, 先分别对 left 和 right 递归调用, 最后在调换 left 与 right 的位置.

```
func invert() -> Node {
    if left == nil && right == nil {
        return self
    }

    if let left = left {
        _ = left.invert()
    }
    if let right = right {
        _ = right.invert()
    }

    let tmp = left
    left = right
    right = tmp

    return self
}
```

[](#二叉树的遍历 "二叉树的遍历")二叉树的遍历
--------------------------

### [](#深度优先遍历-DFS "深度优先遍历(DFS)")深度优先遍历 (DFS)

利用 Stack 的特点 (push/pop) 可以非常方便地实现二叉树的深度优先遍历(Depth First Search).

```
func depthFirstSearch() {
    var stack = [Node]()
    stack.append(self)

    while stack.count > 0 {
        guard let currentNode = stack.last else {
            continue
        }

        currentNode.visit()
        stack.removeLast()

        if let right = currentNode.right {
            stack.append(right)
        }
        if let left = currentNode.left {
            stack.append(left)
        }
    }
}
```

### [](#广度优先遍历-BFS "广度优先遍历(BFS)")广度优先遍历 (BFS)

利用 Queue 的特点 (enqueue/dequeue) 可以非常方便地实现二叉树的广度优先遍历(Breadth First Search).

```
func breadthFirstSearch() {
    var array = [Node]()
    array.append(self)

    while array.count > 0 {
        guard let currentNode = array.first else {
            continue
        }

        currentNode.visit()
        array.removeFirst()

        if let left = currentNode.left {
            array.append(left)
        }
        if let right = currentNode.right {
            array.append(right)
        }
    }
}
```

### [](#前序遍历 "前序遍历")前序遍历

```
func preOrderTraversal() {
    visit()
    if let left = left {
        left.preOrderTraversal()
    }
    if let right = right {
        right.preOrderTraversal()
    }
}
```

用 Stack 来实现前序遍历:

```
func preOrderTraversalViaStack() {
    var stack = [Node]()
    stack.append(self)

    while stack.count > 0 {
        guard let currentNode = stack.last else {
            continue
        }

        currentNode.visit()
        stack.removeLast()

        if let right = currentNode.right {
            stack.append(right)
        }
        if let left = currentNode.left {
            stack.append(left)
        }
    }
}
```

### [](#中序遍历 "中序遍历")中序遍历

```
func middleOrderTraversal() {
    if let left = left {
        left.middleOrderTraversal()
    }
    visit()
    if let right = right {
        right.middleOrderTraversal()
    }
}
```

用 Stack 来实现中序遍历:

```
func middleOrderTraversalViaStack() {
    var stack = [Node]()
    var node: Node? = self

    while stack.count > 0  || node != nil {
        if node != nil {
            stack.append(node!)
            node = node?.left
        } else {
            node = stack.removeLast()
            node?.visit()
            node = node?.right
        }
    }
}
```

### [](#后序遍历 "后序遍历")后序遍历

```
func lastOrderTraversal() {
    if let left = left {
        left.lastOrderTraversal()
    }
    if let right = right {
        right.lastOrderTraversal()
    }
    visit()
}
```

用 Stack 来实现后序遍历:

[](#示例代码 "示例代码")示例代码
--------------------

[swift_algorithm](https://github.com/andyccc/swift_algorithm)

[](#参考 "参考")参考
--------------

[swift-algorithm-club](https://github.com/raywenderlich/swift-algorithm-club)

坚持原创技术分享，您的支持将鼓励我继续创作！ So，来杯咖啡？
