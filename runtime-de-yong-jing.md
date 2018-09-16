# Runtime 的應用場景

Runtime 在對於大型框架的開發中，有許多的應用場景。平時常見的類似：

1. Objective-C Associated Object: 動態添加類別屬性
2. Method Swizzling: 動態添加方法與 KVO
3. JSPatch: 熱更新
4. NSCoding

## Objective-C Associated Object: 動態添加類別屬性

在 Category 裡，添加的屬性不會自動生成 getter 與 setter。（Swift Extension 裡增加屬性也必須自訂 get / set\)

所以如果要動態添加屬性，可藉由關聯物件設置的 API 完成。例如:

```objectivec
//設置關聯物件
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
//取得關聯物件
id objc_getAssociatedObject(id object, const void *key)
//移除關聯物件
void objc_removeAssociatedObjects(id object)
```

舉實際例子來說，假設我們要為 UIView 添加一個預設顏色的屬性，可以這樣做：

```objectivec
#import "ViewController.h"
#import "objc/runtime.h"

@interface UIView (DefaultColor)

@property (nonatomic, strong) UIColor *defaultColor;

@end

@implementation UIView (DefaultColor)

@dynamic defaultColor;

// 對於該屬性的識別符
static char kDefaultColorKey;

- (void)setDefaultColor:(UIColor *)defaultColor {
    objc_setAssociatedObject(self, &kDefaultColorKey, defaultColor, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)defaultColor {
    return objc_getAssociatedObject(self, &kDefaultColorKey);
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    UIView *test = [UIView new];
    // 添加的動態屬性
    test.defaultColor = [UIColor blackColor];
}

@end
```

{% hint style="info" %}
關於設置關聯物件時，要以什麼屬性來設置它的內存管理方式，可以參考之前寫的 [ARC](https://keithlee4.gitbook.io/ios-fundamentals-arc/arc-shi-fu) 文章
{% endhint %}

## Method Swizzling: 動態添加方法與 KVO

在消息傳遞與轉發的步驟裡其實就有提及添加方法的部分，這裡給一個實際的例子，假設我們抽換 viewDidLoad 方法的執行內容，換成 klViewDidLoad，可以這麼做：

```objectivec
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        SEL originalDidLoad = @selector(viewDidLoad);
        SEL klDidLoad = @selector(klViewDidLoad);
        
        Method originalMethod = class_getInstanceMethod(class, originalDidLoad);
        IMP originalIMP = method_getImplementation(originalMethod);
        const char *originalType = method_getTypeEncoding(originalMethod);
        
        Method klMethod = class_getInstanceMethod(class, klDidLoad);
        IMP klIMP = method_getImplementation(klMethod);
        const char *klType = method_getTypeEncoding(klMethod);
        
        BOOL didAddMethod = class_addMethod(class, klDidLoad, klIMP, klType);
        if (didAddMethod) {
            class_replaceMethod(class, klDidLoad, originalIMP, originalType);
        }else {
            method_exchangeImplementations(originalMethod, klMethod);
        }
    });
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

- (void)klViewDidLoad {
    [super viewDidLoad];
    NSLog(@"KLViewDidLoad");
}

```

要注意的是，`swizzling` 應該要在 `+load` 或 `+initialize` 完成 ，在 OC 運行期間，每個類別都有兩個方法會被調用，`+load` 在類別被初始裝載時調用， `+initilialze` 在程序第一次調用該類別的方法前調用。

另外，`swizzling` 應該要搭配 `dispatch_once`，確保改變全局狀態的操作只執行這一次。

### KVO

`KVO` 的實現原理也來自於 `Runtime`，當我們 observe 類別 A 的一個屬性時，實際上發生的事情會包含：

1. `KVO` 動態創建一個 A 的子類別，命名為 `NSKVONotifying_A`。
2. 藉由 `isa-swizzling`，將原本 A 的 `isa` 指向到子類。
3. 在子類覆寫被觀察屬性 `keyPath` 的 `setter`。
4. `setter` 隨後負責通知 observer 屬性的改變。在原調用之前與之後，通知所有觀察者屬性值的更改狀況。

## JSPatch 熱更新

雖然這一兩年 Apple 已經明令不可使用熱更新機制了，但是說到 Runtime 最棒的應用還是不得不提到 JSPatch。JSPatch 利用 Runtime 的動態特性，實現方法添加、替換、自訂消息轉發等一系列功能，是你上線後~~躲避審核~~的好朋友。

相關實現原理詳解可以看[這篇](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)。\(~~還沒看完\)~~

## NSCoding

我們可以使用 runtime 提供的函數遍歷屬性，進行 encode 與 decode 操作。例如：

```objectivec
- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super init]) {
        unsigned int outCount;
        Ivar * ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar ivar = ivars[i];
            NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
            [self setValue:[aDecoder decodeObjectForKey:key] forKey:key];
        }
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int outCount;
    Ivar * ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar ivar = ivars[i];
        NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
        [aCoder encodeObject:[self valueForKey:key] forKey:key];
    }
}
```



以上便是幾個 Runtime 常使用到的應用場景。善加使用 Runtime 提供的動態特性，能夠更有效地為系統添加屬性以及修改方法內容。目前項目上比較常使用關聯物件來為第三方庫添加自定義屬性、以及使用 method swizzling 增添功能進去。

