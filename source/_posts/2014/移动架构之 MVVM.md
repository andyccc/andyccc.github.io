
---
title: 移动架构之 MVVM
date: 2014-07-16 21:22:10
categories: 
- 移动架构
tags:
- 移动架构
- MVVM
---


> MVVM 关于 MVVM 就不多做介绍了, 这里通过一个 demo 来简单展示一下. 与 MVP 类似, View 持有 ViewModel，ViewModel 持有 Model. 最大的不同在于 MVVM 通常与双向绑定机制联系......


[](#MVVM "MVVM")MVVM
--------------------

关于 MVVM 就不多做介绍了, 这里通过一个 demo 来简单展示一下.  
与 MVP 类似, **_View 持有 ViewModel，ViewModel 持有 Model_**.

最大的不同在于 MVVM 通常与双向绑定机制联系在一起. 而 iOS 中可以使用 CocoaReact 或 RxSwift 这些自带双向绑定功能的框架, 也可以自己使用 KVO 实现.

[![][img-0]

### [](#Model "Model")Model

```
@interface ModelUserInfo : NSObject

@property (nonatomic, copy)     NSString    *name;
@property (nonatomic, assign)   NSInteger   age;
@property (nonatomic, copy)     NSString    *city;

@end
```

### [](#View "View")View

```
@interface ViewUserInfo : UIView

@property (nonatomic, strong) UITextField *textFieldName;
@property (nonatomic, strong) UITextField *textFieldAge;
@property (nonatomic, strong) UITextField *textFieldCity;

/**
 View持有ViewModel，ViewModel持有Model
 */
@property (nonatomic, strong) ViewModelUserInfo *viewModelUserInfo;

@end
```

其实现文件如下, 其中通过 KVO 来实现了绑定机制.

```
// context of KVO
static NSInteger ctxKVOName     = 0;
static NSInteger ctxKVOAge      = 1;
static NSInteger ctxKVOCity     = 2;

@implementation ViewUserInfo

- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        self.backgroundColor = [UIColor lightGrayColor];
        
        self.textFieldName = [[UITextField alloc] initWithFrame:CGRectMake(20, 50, 200, 30)];
        self.textFieldName.text = @"name";
        [self addSubview:self.textFieldName];
        
        self.textFieldAge = [[UITextField alloc] initWithFrame:CGRectMake(20, 100, 200, 30)];
        self.textFieldAge.text = @"age";
        [self addSubview:self.textFieldAge];
        
        self.textFieldCity = [[UITextField alloc] initWithFrame:CGRectMake(20, 150, 200, 30)];
        self.textFieldCity.text = @"city";
        [self addSubview:self.textFieldCity];
    }
    return self;
}

- (void)setViewModelUserInfo:(ViewModelUserInfo *)viewModelUserInfo
{
    _viewModelUserInfo = viewModelUserInfo;
    
    // View -> Model
    /**
     Using UserEvent to bind
     */
    [self.textFieldName addTarget:self
                           action:@selector(actionViewTextFieldChanged:)
                 forControlEvents:UIControlEventEditingChanged];
    [self.textFieldAge addTarget:self
                          action:@selector(actionViewTextFieldChanged:)
                forControlEvents:UIControlEventEditingChanged];
    [self.textFieldCity addTarget:self
                           action:@selector(actionViewTextFieldChanged:)
                 forControlEvents:UIControlEventEditingChanged];
    
    
    // Model -> View
    /**
     Using KVO to bind
     */
    [_viewModelUserInfo.modelUserInfo addObserver:self
                     forKeyPath:@"name"
                        options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
                        context:&ctxKVOName];
    [_viewModelUserInfo.modelUserInfo addObserver:self
                     forKeyPath:@"age"
                        options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
                        context:&ctxKVOAge];
    [_viewModelUserInfo.modelUserInfo addObserver:self
                     forKeyPath:@"city"
                        options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionInitial
                        context:&ctxKVOCity];
}

- (void)actionViewTextFieldChanged:(UITextView *)sender
{
    NSLog(@"View -> Model");
    
    if ([sender isEqual:self.textFieldName]) {
        self.viewModelUserInfo.modelUserInfo.name = sender.text;
    } else if ([sender isEqual:self.textFieldAge]) {
        self.viewModelUserInfo.modelUserInfo.age = [sender.text integerValue];
    } else if ([sender isEqual:self.textFieldCity]) {
        self.viewModelUserInfo.modelUserInfo.city = sender.text;
    }
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if (context == &ctxKVOName) {
        
        NSString *newName = change[NSKeyValueChangeNewKey];
        if (![newName isKindOfClass:[NSNull class]] && newName) {
            dispatch_async(dispatch_get_main_queue(), ^{
                self.textFieldName.text = newName;
            });
        }
        
    } else if (context == &ctxKVOAge) {
        
        NSNumber *newAge = change[NSKeyValueChangeNewKey];
        if (![newAge isKindOfClass:[NSNull class]] && newAge) {
            dispatch_async(dispatch_get_main_queue(), ^{
                self.textFieldAge.text = [NSString stringWithFormat:@"%@", newAge];
            });
        }
        
    } else if (context == &ctxKVOCity) {
        
        NSString *newCity = change[NSKeyValueChangeNewKey];
        if (![newCity isKindOfClass:[NSNull class]] && newCity) {
            dispatch_async(dispatch_get_main_queue(), ^{
                self.textFieldCity.text = newCity;
            });
        }
        
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}

@end
```

### [](#ViewModel "ViewModel")ViewModel

```
@interface ViewModelUserInfo : NSObject

@property (nonatomic, strong) ModelUserInfo *modelUserInfo;

/**
 模拟服务端等对Model进行修改
 */
- (void)updateModelFromMockWeb;

@end
```

其实现文件:

```
@implementation ViewModelUserInfo

- (instancetype)init
{
    self = [super init];
    if (self) {
        self.modelUserInfo = [[ModelUserInfo alloc] init];
    }
    return self;
}

- (void)updateModelFromMockWeb
{
    self.modelUserInfo.name   = @"Chris1";
    self.modelUserInfo.age    = 1;
    self.modelUserInfo.city   = @"Shanghai1";
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        self.modelUserInfo.name   = @"Chris18";
        self.modelUserInfo.age    = 18;
        self.modelUserInfo.city   = @"Shanghai18";
    });
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        self.modelUserInfo.name   = @"Chris28";
        self.modelUserInfo.age    = 28;
        self.modelUserInfo.city   = @"iloveShanghai28";
    });
}

@end
```

### [](#如何使用 "如何使用")如何使用

```
@interface ViewController ()

/**
 View持有ViewModel，ViewModel持有Model
 */
@property (nonatomic, strong) ViewUserInfo *viewUserInfo;
@property (nonatomic, strong) ViewModelUserInfo *viewModelUserInfo;

@property (nonatomic, strong) UIButton *btnUpdate;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.view addSubview:self.viewUserInfo];
    self.viewUserInfo.viewModelUserInfo = self.viewModelUserInfo;
    
    [self.view addSubview:self.btnUpdate];
}

- (ViewUserInfo *)viewUserInfo
{
    if (!_viewUserInfo) {
        _viewUserInfo = [[ViewUserInfo alloc] initWithFrame:self.view.bounds];
    }
    
    return _viewUserInfo;
}

- (ViewModelUserInfo *)viewModelUserInfo
{
    if (!_viewModelUserInfo) {
        _viewModelUserInfo = [[ViewModelUserInfo alloc] init];
    }
    
    return _viewModelUserInfo;
}

- (UIButton *)btnUpdate
{
    if (!_btnUpdate) {
        _btnUpdate = [[UIButton alloc] initWithFrame:CGRectMake(100, 250, 100, 50)];
        _btnUpdate.backgroundColor = [UIColor greenColor];
        [_btnUpdate setTitle:@"Update" forState:UIControlStateNormal];
        [_btnUpdate addTarget:self action:@selector(actionUpdate:) forControlEvents:UIControlEventTouchUpInside];
    }
    return _btnUpdate;
}

- (void)actionUpdate:(UIButton *)sender
{
    // Model -> View
    NSLog(@"Model -> View");
    [self.viewModelUserInfo updateModelFromMockWeb];
}

@end
```

[](#单元测试 "单元测试")单元测试
--------------------

```
@interface DemoMVVMTests : XCTestCase

@property (nonatomic, strong) ViewModelUserInfo *viewModel;

@end

@implementation DemoMVVMTests

- (void)setUp {
    [super setUp];
    // Put setup code here. This method is called before the invocation of each test method in the class.
    
    self.viewModel = [[ViewModelUserInfo alloc] init];
}

- (void)tearDown {
    // Put teardown code here. This method is called after the invocation of each test method in the class.
    [super tearDown];
}

- (void)testExample {
    // This is an example of a functional test case.
    // Use XCTAssert and related functions to verify your tests produce the correct results.
    
    [self.viewModel updateModelFromMockWeb];
    
    XCTAssert([self.viewModel.modelUserInfo.name isEqualToString:@"Chris1"], "name等于Chris1");
    XCTAssert(self.viewModel.modelUserInfo.age == 1, "age等于1");
    XCTAssert([self.viewModel.modelUserInfo.city isEqualToString:@"Shanghai1"], "city等于Shanghai1");
    
    sleep(4);
    XCTAssert([self.viewModel.modelUserInfo.name isEqualToString:@"Chris18"], "name等于Chris18");
    XCTAssert(self.viewModel.modelUserInfo.age == 18, "age等于18");
    XCTAssert([self.viewModel.modelUserInfo.city isEqualToString:@"Shanghai18"], "city等于Shanghai18");
    
    sleep(6);
    XCTAssert([self.viewModel.modelUserInfo.name isEqualToString:@"Chris28"], "name等于Chris28");
    XCTAssert(self.viewModel.modelUserInfo.age == 28, "age等于28");
    XCTAssert([self.viewModel.modelUserInfo.city isEqualToString:@"iloveShanghai28"], "city等于iloveShanghai28");
}
@end
```

可以看出, MVVM 下的单元测试确实非常方便, 仅对代码逻辑进行测试即可, 而 UI 一般是事先绑定好的, 所以基本不会出现异常情况.

但缺点也比较明显, 因为数据绑定使得调试不便, 项目复杂的时候, 数据绑定需要额外消耗比较多的内存资源.

[](#示例代码 "示例代码")示例代码
--------------------

[DemoMVVM](https://github.com/andyccc/DemoMVVM)
