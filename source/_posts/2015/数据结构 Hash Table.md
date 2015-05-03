---
title: 数据结构 Hash Table
date: 2015-07-11 10:20:10
categories: 
-数据结构
tags:
- 数据结构
- Hash Table
---


> Hash Table 哈希表是 Dictionary(键值对) 的基础, 其访问及存储对象的基础都是 key-value 的形式. Hashable 在 Swift 中, key 必须遵循 Hashable 协议, 可以对应......


[](#Hash-Table "Hash Table")Hash Table
--------------------------------------

哈希表是 Dictionary(键值对) 的基础, 其访问及存储对象的基础都是 key-value 的形式.

[](#Hashable "Hashable")Hashable
--------------------------------

在 Swift 中, key 必须遵循 Hashable 协议, 可以对应得到一个 hashValue, 该 hashValue 通常是唯一的一个整数值. 如每个 String 对象都能得到一个 hashValue.

```
// 根据Hashable协议, 每个key可以算出一个hashValue, 对应一个index(可能是多对一的关系)
private func p_index(forKey key: Key) -> Int {
    return abs(key.hashValue) % buckets.count
}
```

哈希表内部使用一个数组存储 key-value 对, 所以哈希表的容量就是其内部数组的容量.

[](#操作复杂度 "操作复杂度")操作复杂度
-----------------------

因为 key-value 的存储结构, 哈希表的的查找 / 添加 / 删除等操作, 平均时间复杂度为 O(1).

[](#collisions "collisions")collisions
--------------------------------------

不同的 String 对象, 根据 hashValue 计算得到的 index 可能会相等.  
即一个 index, 可能对应于多个 key-value 键值对, 所以这里采用一个 index 对应一个 key-value 键值对组成的数组.

```
private typealias Element = (key: Key, value: Value)
private typealias Bucket = [Element]

// 可能存在collisions(不同key的hashValue得到的index相等)
// 所以, 每个index对应的Element不止一个. (也使用数组Bucket存储)
private var buckets: [Bucket]
```

[](#Swift实现 "Swift实现")Swift 实现
------------------------------

```
public func value(forKey key: Key) -> Value? {
    let index = p_index(forKey: key)
    // 可能存在collisions而导致不同key的hashValue得到的index相等
    for element in buckets[index] {
        if element.key == key {
            return element.value
        }
    }
    return nil
}

/**
    考虑已存在的情况
*/
@discardableResult
public mutating func updateValue(_ value: Value, forKey key: Key) -> Value? {
    let index = p_index(forKey: key)

    for (idx, element) in buckets[index].enumerated() {
        if element.key == key {
            let oldValue = element.value
            buckets[index][idx].value = value
            return oldValue
        }
    }

    buckets[index].append((key: key, value: value))
    count = count + 1

    return nil
}

@discardableResult
public mutating func removeValue(forKey key: Key) -> Value? {
    let index = p_index(forKey: key)

    for (idx, element) in buckets[index].enumerated() {
        if element.key == key {
            buckets[index].remove(at: idx)
            count = count - 1
            return element.value
        }
    }

    return nil
}

public mutating func removeAll() {
    buckets = Array<Bucket>(repeatElement([], count: buckets.count))
    count = 0
}
```

[](#subscript "subscript")subscript
-----------------------------------

```
// 利用Swift的subscript
public subscript(key: Key) -> Value? {
    get {
        return value(forKey: key)
    }
    set {
        if let value = newValue {
            updateValue(value, forKey: key)
        } else {
            removeValue(forKey: key)
        }
    }
}
```

[](#总结 "总结")总结
--------------

Hash Table 的关键在于 key 值遵循 Hashable 协议.  
另外, 要采取一定措施避免出现 collisions(即一个 index 对应多个 key-value 键值对的情况). 该措施的要点在于 **_根据每个 key 算出的 index, 所对应的是多个 Element 组成的数组_** .

```
public struct HashTable<Key: Hashable, Value> {
    private typealias Element = (key: Key, value: Value)
    private typealias Bucket = [Element]

    // 可能存在collisions(不同key的hashValue得到的index相等)
    // 所以, 每个index对应的Element不止一个. (也使用数组Bucket存储)
    private var buckets: [Bucket]

    // 用于记录buckets中的元素个数(而非每个Bucket中的Element个数)
    private(set) public var count = 0

    public var isEmpty: Bool { return count == 0 }

    public init(capacity: Int) {
        assert(capacity > 0)
        buckets = Array<Bucket>(repeatElement([], count: capacity))
    }

    public func value(forKey key: Key) -> Value? {
        let index = p_index(forKey: key)
        // 可能存在collisions而导致不同key的hashValue得到的index相等
        for element in buckets[index] {
            if element.key == key {
                return element.value
            }
        }
        return nil
    }

    /**
        考虑已存在的情况
    */
    @discardableResult
    public mutating func updateValue(_ value: Value, forKey key: Key) -> Value? {
        let index = p_index(forKey: key)

        for (idx, element) in buckets[index].enumerated() {
            if element.key == key {
                let oldValue = element.value
                buckets[index][idx].value = value
                return oldValue
            }
        }

        buckets[index].append((key: key, value: value))
        count = count + 1

        return nil
    }

    @discardableResult
    public mutating func removeValue(forKey key: Key) -> Value? {
        let index = p_index(forKey: key)

        for (idx, element) in buckets[index].enumerated() {
            if element.key == key {
                buckets[index].remove(at: idx)
                count = count - 1
                return element.value
            }
        }

        return nil
    }

    public mutating func removeAll() {
        buckets = Array<Bucket>(repeatElement([], count: buckets.count))
        count = 0
    }

    // 利用Swift的subscript
    public subscript(key: Key) -> Value? {
        get {
            return value(forKey: key)
        }
        set {
            if let value = newValue {
                updateValue(value, forKey: key)
            } else {
                removeValue(forKey: key)
            }
        }
    }

    // 根据Hashable协议, 每个key可以算出一个hashValue, 对应一个index(可能是多对一的关系)
    private func p_index(forKey key: Key) -> Int {
        return abs(key.hashValue) % buckets.count
    }
}
```

[](#Set与Dictionary "Set与Dictionary")Set 与 Dictionary
----------------------------------------------------

[](#示例代码 "示例代码")示例代码
--------------------

[swift_algorithm](https://github.com/andyccc/swift_algorithm)

[](#参考 "参考")参考
--------------

[swift-algorithm-club](https://github.com/raywenderlich/swift-algorithm-club)

