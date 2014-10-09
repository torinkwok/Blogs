除了调试器，有时候caveman debugging也是很好的调试方法。最近写app的时候，经常想能够在Cocoa框架中的一些方法中插入调试语句，这样在一些我想监视的框架方法被调用时，也可以向console打印调试语句，*这会使得调试变得更加容易。*

举个例子，我使用`NSDate`类创建一个日期对象：
```
NSDate* TongGuosBirthday = [ NSDate dateWithString: @"1997-01-28 19:22:34 +0600" ];
```

我想在`+[ NSDate dateWithString: ]`这个类方法在被调用时能够向console打印调试语句，但是因为`NSDate`是被编译成二进制的系统框架中的类，其内部实现是一个黑盒，我无法向其`dateWithString:`方法的实现添加额外的代码。解决方法一般就是进行子类化，在子类中覆写某一方法，但是因为Cocoa中的一些类是以class cluster（类簇）的方式实现的，很多情况下很难正确进行子类化。那么另一种比较丑陋的解决方案就是以category的形式添加一个其他名称的方法（比如`debug_dateWithSring:`），将其作为`dateWithString:`的cover，同时加入自己的调试语句：

```
// NSDate + GTDebugForNSDate
@interface NSDate ( GTDebugForNSDate )
+ ( instancetype ) debug_dateWithString: ( NSString* )_String;
@end

@implementation NSDate ( GTDebugForNSDate )
+ ( instancetype ) debug_dateWithString: ( NSString* )_String
    {
    if ( [ _String isEqualToString: @"1997-01-28 19:22:34 +0600" ] )
        NSLog( @"Happy birthday, Tong Guo!" );

    return [ self dateWithString: _String ];
    }
@end // NSDate + GTDebugForNSDate
```

然后在所有使用`dateWithString:`的地方使用`debug_dateWithString:`，但是我想要一种更优雅的方式，可以像平常一样调用框架中的方法，同时还能够插入自己的代码，这个时候Objective-C作为一门动态语言的威力就展现出来了。

Objective-C对象在收到消息之后，在运行时才会进行解析应该调用哪个方法，利用其动态语言的优势，我们可以只用一种称作"method swizzling"的技巧，在不需要源代码也不需要子类化的情况下，“悄无声息”地替换掉黑盒方法的实现。

这首先需要了解一点背景：
Objective-C的类的方法列表会把selector的名称映射到相关的方法的实现上，动态消息派发系统会据此找到应该调用的方法。这些方法的实现都是以IMP类型的函数指针来表示的，IMP其实是一个函数指针的typedef，其原型如下：
```
id  ( *IMP )( id,  SEL, ... )
```

`NSDate`可以响应`+ dateWithNaturalLanguageString:`,`+ dateWithString:`，`+ distantFuture`等selector，这些selector都被映射到各自IMP上。

这其实是一张由每个类维护的表格，映射selector到各自的IMP上，这张表格本质上是可读写的，用什么读写呢？Objective-C在运行时系统中提供的很多纯C函数就用于操作这张表，我们可以用这些函数向这张表格中新增selector，可以改变某个selector所对应的方法实现。甚至，还可以交换两个selector所映射到的IMP指针。

在实现method swizzling之前，先来看看如何交换两个selector所映射到的IMP指针，说通俗点，就是怎样在运行时互换两个已经写好的方法实现。相交换方法实现，可以使用<objc/runtime.h>中的：
```
void method_exchangeImplementations( Method _lhsMethod, Method _rhsMethod );
```
该函数的两个参数表示待交换的两个方法的实现，方法实现可以通过
```
Method class_getClassMethod( Class _Class, SEL _Selector );
Method class_getInstanceMethod( Class _Class, SEL _Selector );
```
分别获得，前者用于获取类中类方法（class method）的方法实现，后者用于获取类中实例方法（instance method）的方法实现。

举个`method_exchangeImplementations()`函数的使用方法的例子，比如我们想交换`NSString`类中`lowercaseString`和`uppercaseString`方法的实现，即正常情况下，`lowercaseString`方法将字符串对象中的全部字符转换为小写形式，而`uppercaseString`方法将字符串中的全部字符转换为大写形式，交换它们的方法实现，就意味着`lowercaseString`将会将字符串中的所有字符转换为大写形式，而`uppercaseString`会将字符串中的所有字符转换为小写形式：

```
Method instanceMethod_lowercaseString = class_getInstanceMethod( [ NSString class ], @selector( lowercaseString ) );

Method instanceMethod_uppercaseString = class_getInstanceMethod( [ NSString class ], @selector( uppercaseString ) );

NSString* hello = @"Hello, WORLD!";

// Before exchanging
NSLog( @"Before exchanging:" );
NSLog( @"%@", [ hello lowercaseString ] );
NSLog( @"%@", [ hello uppercaseString ] );
printf( "\n\n" );

// Begin to exchange
method_exchangeImplementations( instanceMethod_lowercaseString, instanceMethod_uppercaseString  );

// After exchanging
NSLog( @"After exchanging:" );
NSLog( @"%@", [ hello lowercaseString ] );
NSLog( @"%@", [ hello uppercaseString ] );
printf( "\n\n" );
```

比较一下交换实现前后的输出结果：
```
2014-10-09 01:56:05.665 Term3[75744:303] Before exchanging:
2014-10-09 01:56:05.667 Term3[75744:303] hello, world!
2014-10-09 01:56:05.667 Term3[75744:303] HELLO, WORLD!


2014-10-09 01:56:05.667 Term3[75744:303] After exchanging:
2014-10-09 01:56:05.668 Term3[75744:303] HELLO, WORLD!
2014-10-09 01:56:05.668 Term3[75744:303] hello, world!
```

没错，我们在没有源代码并且也不进行子类化的情况下，利用Objective-C强大的动态特性成功地在运行时“悄无声息”地替换了黑盒方法的实现。

这个例子有些~~丧心病狂~~干得漂亮。这只是为了演示一下`method_exchangeImplementations`的效果，一般这种直接交换是毫无意义的，但是如果稍加改动，就可以通过这一手段来实现method swizzling，为即有的黑盒方法添加新功能。

现在回到开头我们遇到的关于向`NSDate`的`dateWithString:`方法中添加调试语句的情况，可以利用这种手段实现的method swizzling技巧轻松做到：

首先，保持之前编写的`GTDebugForNSDate`分类和`debug_dateWithString:`方法：
但是实现部分稍加更改：

```
// NSDate + GTDebugForNSDate
@interface NSDate ( GTDebugForNSDate )
+ ( instancetype ) debug_dateWithString: ( NSString* )_String;
@end

@implementation NSDate ( GTDebugForNSDate )
+ ( instancetype ) debug_dateWithString: ( NSString* )_String
    {
    if ( [ _String isEqualToString: @"1997-01-28 19:22:34 +0600" ] )
        NSLog( @"Happy birthday, Tong Guo!" );

    // Different from prior implementation...
    return [ self debug_dateWithString: _String ];
    }
@end // NSDate + GTDebugForNSDate
```
上边这段代码看上去貌似是会导致infinite loop的递归调用，但实际上，我们之后会在运行时调用该方法之前交换`debug_dateWithString:`与`NSDate`内部的`dateWithString:`方法的实现，所以在上面代码中的

```
return [ self debug_dateWithString: _String ];
```
这行代码实际上会调用`dateWithString:`的实现，而非递归调用。而`debug_dateWithString:`方法本身实际上响应的是`dateWithString:`这个selector。

下面开始调换它们的实现：

```
int main( int _Argc, char* const _Argv[] )
    {
    @autoreleasepool
        {
        Method classMethod__dateWithString = class_getClassMethod( [ NSDate class ], @selector( dateWithString: ) );
        Method classMethod__debug_dateWithString = class_getClassMethod( [ NSDate class ], @selector( debug_dateWithString: ) );

        method_exchangeImplementations( classMethod__dateWithString, classMethod__debug_dateWithString );
        }

    return 0;
    }
```

这时`NSDate`的`dateWithString:`与我们自定义的`NSDate+GTDebugForDate`category中的`debug_dateWithString:`的实现已经互换，在main函数中测试一下：

```
NSDate* TongGuosBirthday = [ NSDate dateWithString: @"1997-01-28 19:22:34 +0600" ];
NSLog( @"%@", TongGuosBirthday );
```

代码在运行时会向console打印如下内容：

```
2014-10-09 14:05:48.891 Term3[28347:303] Happy birthday, Tong Guo!
2014-10-09 14:05:48.898 Term3[28347:303] 1997-01-28 13:22:34 +0000
```

没错，那句`Happy birthday, Tong Guo!`正是在`NSDate`的`dateWithString:`调用时打印的！我们已经成功地向黑盒方法中添加了自己的代码。这就是method swizzling技巧。

利用method swizzling，开发者可以为那些”完全不知道具体实现的（completely opaque）黑盒方法增添log功能，这非常有助于调试！但是目前为止我只发现了其在调试中的作用，很少有情况需要在调试程序之外的场合中互换方法的实现。

> 你问我Objective-C支不支持运行时交换方法实现，我说支持，我就明确告诉你这一点。但是什么时候用，也得按照实际情况来，对不对？你不能因为Objective-C中有这个特性，就一定要使用，明白这个意思吗？你们不要总想着这样很酷，就到处用，naive!