# Runtime 的消息傳遞與轉發

## 消息傳遞機制

舉個簡單的例子: `[obj foo]`，經由 compliler 會轉成消息發送 `objc_msgSend(obj, foo)`.

`objc_msgSend` 的定義為:

`OBJC_EXPORT id objc_msgSend(id self, SEL op, ...)`

 內部執行流程大致是這樣的:

1. 通過 `obj` 的 `isa` 指針找到它的 `class`
2. 在 `class` 的 `cache` 或者 `method list` 找 `foo`
3. 如果找不到，往 `superclass` 找
4. 一旦找到 `foo`，執行它的 `IMP`

如果整個繼承樹搜尋完畢還無法完成消息轉發，系統變化執行 `doesNotRecognizeSelector:` 丟出 `unrecognized selector` 錯誤。

## 消息轉發

方才提到，如果對於一則消息，繼承樹搜尋完畢仍無法找到對應的方法，便會執行消息轉發機制。轉發的完整流程分為三個步驟：

1. 動態方法解析
2. 備用接收者
3. 完整消息轉發

流程圖如下

![](.gitbook/assets/msg_send_flow.txt)

### 動態方法解析

第一步，Objective-C 運行時會調用 `+resolveInstanceMethod:` 或者 `+resolveClassMethod:` 。在這裡我們會有第一次提供自定函數實現的機會。如果返回 YES，系統便會重新啟動消息發送流程。

舉個例子

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //呼叫一個未定義的 Method
    [self performSelector:@selector(doSth:)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(doSth:)) {
        //如果是要執行 doSth, 執行動態解析，指定自定義的 IMP
        //為此類別設定 doSth 的執行函述內容為 myMethod
        class_addMethod([self class], sel, (IMP)myMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

void myMethod(id obj, SEL _cmd) {
    NSLog(@"Do something");
}
```

通過例子可以發現，我們並沒有定義 `doSth:` 函數，但是通過 `class_addMethod` 動態添加 `myMethod` 函數，並且執行該函數的 `IMP`。

如果方法返回的是 `NO`，便會進行下一步。

### 備用接收者

如果目標物件實現 `-forwardingTargetForSelector:` ，Runtime 就會調用該方法。提供我們將消息轉發到其他物件的機會。舉例來說：

```objectivec
#import "ViewController.h"
#import "objc/runtime.h"

@interface Person: NSObject

@end

@implementation Person

- (void)myMethod {
    NSLog(@"Do something");
}

@end

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //呼叫一個未定義的 Method
    [self performSelector:@selector(doSth:)];
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(doSth:)) {
        return [Person new];//返回 Person 物件，由 Person 處理該消息
    }
    
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

### 完整消息轉發

如果上一步還不能處理消息，最後一步就是觸發完整消息轉發機制。首先會發送 `-methodSignatureForSelector:`消息獲得函數的參數和返回值類型。如果`-methodSignatureForSelector:`返回 `nil` ，`Runtime` 則會發出 ~~`-doesNotRecognizeSelector:`~~ 消息，造成程序崩潰。如果返回了一個函數簽名，`Runtime` 就會創建一個 `NSInvocation` 對象發送 `-forwardInvocation:` 消息給目標對象。舉例如下：

```objectivec
#import "ViewController.h"
#import "objc/runtime.h"

@interface Person: NSObject

@end

@implementation Person

- (void)myMethod {
    NSLog(@"Do something");
}

@end

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //呼叫一個未定義的 Method
    [self performSelector:@selector(doSth:)];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"doSth"]) {
        //進行簽名以及傳入簽名參數
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;

    Person *p = [Person new];
    if([p respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:p];
    }
    else {
        [self doesNotRecognizeSelector:sel];
    }

}

@end
```

這裡便進行了完整的轉發，通過簽名，`Runtime` 生成一個 `anInvocation` 物件，發送給 `forwardInvocation` 方法裡的 `Person` 物件去執行該函數。簽名參數的種類可以參照[蘋果官方文件](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)。

簡述完轉發流程後，來介紹一些日常的 `Runtime` 應用。

