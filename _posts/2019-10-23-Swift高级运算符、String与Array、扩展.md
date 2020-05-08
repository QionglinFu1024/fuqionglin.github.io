---
layout: post
title: 'Swift5.0 (十四)'
subtitle: '高级运算符、String与Array、扩展'
date: 2019-10-23
categories: 技术
cover: 
tags: Swift


---

### 关于String的思考

- 1个String变量占用多少内存?
- 下面2个String变量，底层存储有什么不同?

```
var str1 = "0123456789"
var str2 = "0123456789ABCDEF"
```

- 如果对String进行拼接操作， String变量的存储会发生什么变化?

```
str1.append("ABCDE")
str1.append("F")
str2.append("G")
```

- [ASCII码表](https://www.ascii-code.com/)
- 内存地址从低到高

|      代码区      |
| :--------------: |
|      常量区      |
| 全局区（数据段） |
|      堆空间      |
|      栈空间      |
|      动态库      |

---

### 从编码到启动APP

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/Swift5/000033.jpg)

---

### dyld_stub_binder

- 符号的延迟绑定通过dyld_stub_binder完成
- jmpq *0xb31(%rip)格式的汇编指令 
  - 占用6个字节

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/Swift5/000034.jpg)

----

### 关于Array的思考

```
public struct Array<Element>
var arr = [1, 2, 3, 4]
```

- 1个Array变量占用多少内存? n 数组中的数据存放在哪里?

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/Swift5/000035.jpg)

-----

### 溢出运算符(Overflow Operator)

- Swift的算数运算符出现溢出时会抛出运行时错误
-  Swift有溢出运算符(`&+、&-、&*`)，用来支持溢出运算

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/Swift5/000036.jpg)

----

### 运算符重载(Operator Overload)

- 类、结构体、枚举可以为现有的运算符提供自定义的实现，这个操作叫做:运算符重载

```
struct Point {
    var x: Int, y: Int
}

func + (p1: Point, p2: Point) -> Point {
    Point(x: p1.x + p2.x, y: p1.y + p2.y)
}

let p = Point(x: 10, y: 20) + Point(x: 11, y: 22)
print(p) // Point(x: 21, y: 42)

struct Point {
    var x: Int, y: Int
    static func + (p1: Point, p2: Point) -> Point {
        Point(x: p1.x + p2.x, y: p1.y + p2.y)
    }
}
```

```
static func + (p1: Point, p2: Point) -> Point {
    Point(x: p1.x + p2.x, y: p1.y + p2.y)
}

static func - (p1: Point, p2: Point) -> Point {
    Point(x: p1.x - p2.x, y: p1.y - p2.y)
}

static prefix func - (p: Point) -> Point {
    Point(x: -p.x, y: -p.y)
}

static func += (p1: inout Point, p2: Point) {
    p1 = p1 + p2
}

static prefix func ++ (p: inout Point) -> Point {
    p += Point(x: 1, y: 1)
    return p
}

static postfix func ++ (p: inout Point) -> Point {
    let tmp = p
    p += Point(x: 1, y: 1)
    return tmp
}

static func == (p1: Point, p2: Point) -> Bool {
    (p1.x == p2.x) && (p1.y == p2.y)
}
```

---

### Equatable

- 要想得知2个实例是否等价，一般做法是遵守`Equatable` 协议，重载== 运算符 
  - 与此同时，等价于重载了 != 运算符
- Swift为以下类型提供默认的`Equatable` 实现 
  - 没有关联类型的枚举
  - 只拥有遵守 `Equatable` 协议关联类型的枚举 
  - 只拥有遵守 `Equatable` 协议存储属性的结构体
- 引用类型比较存储的地址值是否相等(是否引用着同一个对象)，使用恒等运算符=== 、!==

```
struct Point : Equatable {
    var x: Int, y: Int
}
var p1 = Point(x: 10, y: 20)
var p2 = Point(x: 11, y: 22)
print(p1 == p2) // false
print(p1 != p2) // true
```

---

### Comparable

- 要想比较2个实例的大小，一般做法是: 
  - 遵守 Comparable 协议 
  - 重载相应的运算符

```
// score大的比较大，若score相等，age小的比较大
struct Student : Comparable {
    var age: Int
    var score: Int
    init(score: Int, age: Int) {
        self.score = score
        self.age = age
    }
    static func < (lhs: Student, rhs: Student) -> Bool {
        (lhs.score < rhs.score) || (lhs.score == rhs.score && lhs.age > rhs.age)
    }
    static func > (lhs: Student, rhs: Student) -> Bool {
        (lhs.score > rhs.score) || (lhs.score == rhs.score && lhs.age < rhs.age)
    }
    static func <= (lhs: Student, rhs: Student) -> Bool {
        !(lhs > rhs)
    }
    static func >= (lhs: Student, rhs: Student) -> Bool {
        !(lhs < rhs)
    }
}


var stu1 = Student(score: 100, age: 20)
var stu2 = Student(score: 98, age: 18)
var stu3 = Student(score: 100, age: 20)
print(stu1 > stu2) // true
print(stu1 >= stu2) // true
print(stu1 >= stu3) // true
print(stu1 <= stu3) // true
print(stu2 < stu1) // true
print(stu2 <= stu1) // true
```

---

### 自定义运算符(Custom Operator)

- 可以自定义新的运算符:在全局作用域使用operator进行声明
- Apple文档参考: [链接1](https://developer.apple.com/documentation/swift/swift_standard_library/operator_declarations) [链接2](https://docs.swift.org/swift- book/ReferenceManual/Declarations.html#ID380)
- phttps://docs.swift.org/swift- book/ReferenceManual/Declarations.html#ID380

```
prefix operator 前缀运算符
postfix operator 后缀运算符
infix operator 中缀运算符 : 优先级组

precedencegroup 优先级组 {
    associativity: 结合性(left\right\none)
    higherThan: 比谁的优先级高
    lowerThan: 比谁的优先级低
    assignment: true代表在可选链操作中拥有跟赋值运算符一样的优先级
}

prefix operator +++
infix operator +- : PlusMinusPrecedence
precedencegroup PlusMinusPrecedence {
    associativity: none
    higherThan: AdditionPrecedence
    lowerThan: MultiplicationPrecedence
    assignment: true
}
```

```
struct Point {
    var x: Int, y: Int
    static prefix func +++ (point: inout Point) -> Point {
        point = Point(x: point.x + point.x, y: point.y + point.y)
        return point
    }
    static func +- (left: Point, right: Point) -> Point {
        return Point(x: left.x + right.x, y: left.y - right.y)
    }
    static func +- (left: Point?, right: Point) -> Point {
        print("+-")
        return Point(x: left?.x ?? 0 + right.x, y: left?.y ?? 0 - right.y) }
}

struct Person {
    var point: Point
}
var person: Person? = nil
person?.point +- Point(x: 10, y: 20)
```

---

### 扩展(Extension)

- Swift中的扩展，有点类似于OC中的分类(Category) 
-  扩展可以为枚举、结构体、类、协议添加新功能
  - 可以添加方法、计算属性、下标、(便捷)初始化器、嵌套类型、协议等等
- 扩展不能办到的事情
  - 不能覆盖原有的功能 
  - 不能添加存储属性，不能向已有的属性添加属性观察器 
  - 不能添加父类 
  - 不能添加指定初始化器，不能添加反初始化器
  - ...

----

### 计算属性、下标、方法、嵌套类型

```
extension Double {
    var km: Double { self * 1_000.0 }
    var m: Double { self }
    var dm: Double { self / 10.0 }
    var cm: Double { self / 100.0 }
    var mm: Double { self / 1_000.0 }
}

extension Array {
    subscript(nullable idx: Int) -> Element? {
        if (startIndex..<endIndex).contains(idx) {
            return self[idx]
        }
        return nil
    }
}

extension Int {
    func repetitions(task: () -> Void) {
        for _ in 0..<self { task() }
    }
    mutating func square() -> Int {
        self = self * self
        return self
    }
    enum Kind { case negative, zero, positive }
    var kind: Kind {
        switch self {
        case 0: return .zero
        case let x where x > 0: return .positive
        default: return .negative
        }
    }
    subscript(digitIndex: Int) -> Int {
        var decimalBase = 1
        for _ in 0..<digitIndex { decimalBase *= 10 }
        return (self / decimalBase) % 10
    }
}
```

----

### 协议、初始化器

- 如果希望自定义初始化器的同时，编译器也能够生成默认初始化器 
  - 可以在扩展中编写自定义初始化器
-  `required`初始化器也不能写在扩展中

```
class Person {
    var age: Int
    var name: String
    init(age: Int, name: String) {
        self.age = age
        self.name = name
    }
}
extension Person : Equatable {
    static func == (left: Person, right: Person) -> Bool {
        left.age == right.age && left.name == right.name
    }
    convenience init() {
        self.init(age: 0, name: "")
    }
}

struct Point {
    var x: Int = 0
    var y: Int = 0
}
extension Point {
    init(_ point: Point) {
        self.init(x: point.x, y: point.y)
    }
}
var p1 = Point()
var p2 = Point(x: 10)
var p3 = Point(y: 20)
var p4 = Point(x: 10, y: 20)
var p5 = Point(p4)
```

----

### 协议

- 如果一个类型已经实现了协议的所有要求，但是还没有声明它遵守了这个协议 
  - 可以通过扩展来让它遵守这个协议

```
protocol TestProtocol {
    func test()
}
class TestClass {
    func test() {
        print("test")
    }
}
extension TestClass : TestProtocol {}
```

- 编写一个函数，判断一个整数是否为奇数?

```
func isOdd<T: BinaryInteger>(_ i: T) -> Bool {
    i % 2 != 0
}

extension BinaryInteger {
    func isOdd() -> Bool { self % 2 != 0 }
}
```

- 扩展可以给协议提供默认实现，也间接实现『可选协议』的效果 
-  扩展可以给协议扩充『协议中从未声明过的方法』

```
protocol TestProtocol {
    func test1()
}
extension TestProtocol {
    func test1() {
        print("TestProtocol test1")
    }
    func test2() {
        print("TestProtocol test2")
    }
}


class TestClass : TestProtocol {}
var cls = TestClass()
cls.test1() // TestProtocol test1
cls.test2() // TestProtocol test2
var cls2: TestProtocol = TestClass()
cls2.test1() // TestProtocol test1
cls2.test2() // TestProtocol test2



class TestClass : TestProtocol {
    func test1() { print("TestClass test1") }
    func test2() { print("TestClass test2") }
}
var cls = TestClass()
cls.test1() // TestClass test1
cls.test2() // TestClass test2
var cls2: TestProtocol = TestClass()
cls2.test1() // TestClass test1
cls2.test2() // TestProtocol test2
```

----

### 泛型

```
class Stack<E> {
    var elements = [E]()
    func push(_ element: E) {
        elements.append(element) }
    func pop() -> E { elements.removeLast() }
    func size() -> Int { elements.count } }
// 扩展中依然可以使用原类型中的泛型类型
extension Stack {
    func top() -> E { elements.last! }
}
// 符合条件才扩展
extension Stack : Equatable where E : Equatable {
    static func == (left: Stack, right: Stack) -> Bool {
        left.elements == right.elements
    }
}
```

