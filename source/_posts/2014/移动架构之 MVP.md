
---
title: 移动架构之 MVP
date: 2014-06-22 22:11:10
categories: 
- 移动架构
tags:
- 移动架构
- MVP
---

> MVP 关于 MVP 就不多做介绍了, 这里通过一个 demo 来简单展示一下. 关键在于: Presenter 用于连接 View 和 Model, 而 View 自身暴露一个指定方法用于更新 UI. 即 View 拥有 Presen......


[](#MVP "MVP")MVP
-----------------

关于 MVP 就不多做介绍了, 这里通过一个 demo 来简单展示一下.  
关键在于:  
Presenter 用于连接 View 和 Model, 而 View 自身暴露一个指定方法用于更新 UI.  
即 **_View 拥有 Presenter, Presenter 拥有 Model_**.

[![][img-0]

### [](#Model "Model")Model

```
// MARK: - Model

protocol Presentable {
    var name: String { get }
    var age: Int { get }
    
    var addtionalInfo: String { get }
}

struct User: Presentable {
    var name: String
    var age: Int
    
    var addtionalInfo: String {
        return city
    }
    
    let city: String
}

struct Dog: Presentable {
    var name: String
    var age: Int
    
    var addtionalInfo: String {
        return owner
    }
    
    let owner: String
}
```

这里, 通过 User 和 Dog 的对比 (主要是 addtionalInfo 属性), 来区分 Presenter 中传递不同 model 对象的场景.

### [](#View "View")View

```
// MARK: - View

// object to present something (such as UIView)
protocol PresenterDelegate: class {
    func present()
}

// View拥有Presenter
class MyView: UIView, PresenterDelegate {
    var presenter: Presenter!
    
    var lbInfo: UILabel!
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        self.backgroundColor = UIColor.black
        
        self.lbInfo = UILabel(frame: CGRect(x: 0, y: 0, width: 200, height: 50))
        self.lbInfo.font = UIFont.systemFont(ofSize: 20)
        self.lbInfo.textColor = UIColor.white
        self.addSubview(self.lbInfo)
        self.lbInfo.center = self.center
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    func present() {
        self.lbInfo.text = presenter.presentInfo
    }
}
```

### [](#Presenter "Presenter")Presenter

```
// MARK: - Presenter

// Presenter用于连接View和Model, 而View自身暴露一个指定方法用于更新UI.
// Presenter拥有Model
class Presenter {
    // object need to and can be presented (such as model object)
    let presentable: Presentable
    
    required init(presentable: Presentable) {
        self.presentable = presentable
    }
    
    var presentInfo: String {
        return "\(presentable.name), \(presentable.age), \(presentable.addtionalInfo)"
    }
}
```

### [](#如何使用 "如何使用")如何使用

```
class ViewController: UIViewController {
    var myView: MyView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        myView = MyView(frame: self.view.frame)
        view.addSubview(myView)
        
        let user = User(name: "Chris", age: 18, city: "Shanghai")
        let presenter = Presenter(presentable: user)
        myView.presenter = presenter
        myView.present()
    }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        
        DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
            let dog = Dog(name: "Doggee", age: 2, owner: "Chris")
            let presenter = Presenter(presentable: dog)
            self.myView.presenter = presenter
            self.myView.present()
        }
    }
}
```

通过以上实例可以看出, 引入 Presenter 之后, Model, View 与 Presenter 三者的关系非常明确, 且 Model 与 View 二者是完全隔离的, 不会像 MVC 那样有非常多的交叉.

[](#单元测试 "单元测试")单元测试
--------------------

```
class DemoMVPTests: XCTestCase {
    
    var myView: MyView!
    
    var presenter: Presenter!
    
    var user: User!
    var dog: Dog!
    
    override func setUp() {
        super.setUp()
        
        myView = MyView(frame: CGRect(x: 0, y: 0, width: 100, height: 50))
        
        user = User(name: "Chris", age: 18, city: "Shanghai")
        dog = Dog(name: "Doggee", age: 2, owner: "Chris")
        
        presenter = Presenter(presentable: user)
        
        myView.presenter = presenter
    }
    
    override func tearDown() {
        // Put teardown code here. This method is called after the invocation of each test method in the class.
        super.tearDown()
    }
    
    func testExample() {
        myView.present()
        
        var info = "\(user.name), \(user.age), \(user.city)"
        
        XCTAssert(myView.lbInfo.text == info, "myView中展示user信息")
        
        user = User(name: "Chris", age: 20, city: "Shanghai")
        presenter = Presenter(presentable: user)
        myView.presenter = presenter
        myView.present()
        XCTAssert(myView.lbInfo.text != info, "myView中展示信息已修改")
        
        info = "\(dog.name), \(dog.age), \(dog.owner)"
        presenter = Presenter(presentable: dog)
        myView.presenter = presenter
        myView.present()
        XCTAssert(myView.lbInfo.text == info, "myView中展示dog信息")
    }   
}
```

[](#示例代码 "示例代码")示例代码
--------------------

[DemoMVP](https://github.com/andyccc/DemoMVP)
