# RunTime
什么是Runtime

我们写的代码在程序运行过程中都会被转化成runtime的C代码执行，例如[target doSomething];会被转化成objc_msgSend(target,

@selector(doSomething));。

OC中一切都被设计成了对象，我们都知道一个类被初始化成一个实例，这个实例是一个对象。实际上一个类本质上也是一个对象，在runtime中用结构体表

示。

1:获取列表

有时候会有这样的需求，我们需要知道当前类中每个属性的名字（比如字典转模型，字典的Key和模型对象的属性名字不匹配）。

我们可以通过runtime的一系列方法获取类的一些信息（包括属性列表，方法列表，成员变量列表，和遵循的协议列表）。

2:方法调用

如果用实例对象调用实例方法，会到实例的isa指针指向的对象（也就是类对象）操作。

如果调用的是类方法，就会到类对象的isa指针指向的对象（也就是元类对象）中操作。

(1)首先，在相应操作的对象中的缓存方法列表中找调用的方法，如果找到，转向相应实现并执行。

(2)如果没找到，在相应操作的对象中的方法列表中找调用的方法，如果找到，转向相应实现执行

如果没找到，去父类指针所指向的对象中执行1，2.

以此类推，如果一直到根类还没找到，转向拦截调用。

如果没有重写拦截调用的方法，程序报错。

以上的过程给我带来的启发：

重写父类的方法，并没有覆盖掉父类的方法，只是在当前类对象中找到了这个方法后就不会再去父类中找了。

如果想调用已经重写过的方法的父类的实现，只需使用super这个编译器标识，它会在运行时跳过在当前的类对象中寻找方法的过程。

3:拦截调用

在方法调用中说到了，如果没有找到方法就会转向拦截调用。

那么什么是拦截调用呢。

拦截调用就是，在找不到调用的方法程序崩溃之前，你有机会通过重写NSObject的四个方法来处理。

+ (BOOL)resolveClassMethod:(SEL)sel;

+ (BOOL)resolveInstanceMethod:(SEL)sel;
//后两个方法需要转发到其他的类处理

- (id)forwardingTargetForSelector:(SEL)aSelector;

- (void)forwardInvocation:(NSInvocation *)anInvocation;

第一个方法是当你调用一个不存在的类方法的时候，会调用这个方法，默认返回NO，你可以加上自己的处理然后返回YES。

第二个方法和第一个方法相似，只不过处理的是实例方法。

第三个方法是将你调用的不存在的方法重定向到一个其他声明了这个方法的类，只需要你返回一个有这个方法的target。

第四个方法是将你调用的不存在的方法打包成NSInvocation传给你。做完你自己的处理后，调用invokeWithTarget:方法让某个target触发这个方法。

4:动态添加方法

重写了拦截调用的方法并且返回了YES，我们要怎么处理呢？

有一个办法是根据传进来的SEL类型的selector动态添加一个方法。

首先从外部隐式调用一个不存在的方法：
//隐式调用方法
[target performSelector:@selector(resolveAdd:) withObject:@"test"];

然后，在target对象内部重写拦截调用的方法，动态添加方法。

void runAddMethod(id self, SEL _cmd, NSString *string){
    NSLog(@"add C IMP ", string);
}
+ (BOOL)resolveInstanceMethod:(SEL)sel{

    //给本类动态添加一个方法
    
    if ([NSStringFromSelector(sel) isEqualToString:@"resolveAdd:"]) {
    
        class_addMethod(self, sel, (IMP)runAddMethod, "v@:*");
    
    }
    
    return YES;
}
5:关联对象

现在你准备用一个系统的类，但是系统的类并不能满足你的需求，你需要额外添加一个属性。

这种情况的一般解决办法就是继承。

但是，只增加一个属性，就去继承一个类，总是觉得太麻烦类。

这个时候，runtime的关联属性就发挥它的作用了。

6:法交换

方法交换，顾名思义，就是将两个方法的实现交换。例如，将A方法和B方法交换，调用A方法的时候，就会执行B方法中的代码，反之亦然。

话不多说，这是参考Mattt大神在NSHipster上的文章自己写的代码。

