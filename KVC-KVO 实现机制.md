## KVC/KVO 实现机制

### 1、kVO的实现原理

#### 面试题

	kVO 相关：
	1、iOS用什么放肆来实现对一个对象的KVO？（kOV的本质是什么）
	2、如何手动触发kVO？
	3、直接修改成员变量会触发kVO吗？
	
	kVC 相关：
	1、通过kVC修改属性会触发kVO吗？
	2、kVC的赋值和取值过程是怎么样的？原理是什么？
	
### 什么是KVO

kVO的全称是key-Value-Observing，即键值观察。

我们可以监听对象属性值的改变通知观察者。

### KVO简单的实现

	///> DLPerson.h 文件
	
	#import <Foundation/Foundation.h>
	
	NS_ASSUME_NONNULL_BEGIN
	
	@interface DLPerson : NSObject
	
	@property (nonatomic, assign) int age;
	
	@end
	
	NS_ASSUME_NONNULL_END

.m文件	

	///> ViewController.m 文件

	#import "ViewController.h"
	#import "DLPerson.h"
	@interface ViewController ()
	@property (nonatomic, strong) DLPerson *person1;
	@property (nonatomic, strong) DLPerson *person2;
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    self.person1 = [[DLPerson alloc]init];
	    self.person1.age = 1;
	    
	    self.person2 = [[DLPerson alloc]init];
	    self.person2.age = 2;
	    
	    ///> person1添加kvo监听
	    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
	    [self.person1 addObserver:self forKeyPath:@"age" options:options context:nil];
	}
	
	- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
	    self.person1.age = 20;
	    self.person2.age = 30;
	}
	
	- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
	    NSLog(@"监听到了%@的%@属性发生了改变%@",object,keyPath,change);
	}
	
	- (void)dealloc{
	    ///> 使用结束后记得移除
	    [self.person1 removeObserver:self forKeyPath:@"age"];
	}
	
	@end
	
	@end
### 手动调用KVO

KVO在属性发生改变时的调用是自动的，如果想要手动控制这个调用时机，或想自己实现KVO属性的调用，则可以通过KVO提供的方法进行调用。

	- (void)setBalance:(double)theBalance {
	    if (theBalance != _balance) {
	        [self willChangeValueForKey:@"balance"];
	        _balance = theBalance;
	        [self didChangeValueForKey:@"balance"];
	    }
	}

### 实现原理

KVO是通过isa-swizzling技术实现的(这句话是整个KVO实现的重点)。在运行时根据原类创建一个中间类，这个中间类是原类的子类，并动态修改当前对象的isa指向中间类。并且将class方法重写，返回原类的Class。所以苹果建议在开发中不应该依赖isa指针，而是通过class实例方法来获取对象类型。

### 测试代码
为了测试KVO的实现方式，我们加入下面的测试代码。首先创建一个KVOObject类，并在里面加入两个属性，然后重写description方法，并在内部打印一些关键参数。

	@interface KVOObject : NSObject
	@property (nonatomic, copy  ) NSString *name;
	@property (nonatomic, assign) NSInteger age;
	@end
	
	@implementation KVOObject
	
	- (NSString *)description {
	    NSLog(@"object address : %p \n", self);
	    
	    IMP nameIMP = class_getMethodImplementation(object_getClass(self), @selector(setName:));
	    IMP ageIMP = class_getMethodImplementation(object_getClass(self), @selector(setAge:));
	    NSLog(@"object setName: IMP %p object setAge: IMP %p \n", nameIMP, ageIMP);
	    
	    Class objectMethodClass = [self class];
	    Class objectRuntimeClass = object_getClass(self);
	    Class superClass = class_getSuperclass(objectRuntimeClass);
	    NSLog(@"objectMethodClass : %@, ObjectRuntimeClass : %@, superClass : %@ \n", objectMethodClass, objectRuntimeClass, superClass);
	    
	    NSLog(@"object method list \n");
	    unsigned int count;
	    Method *methodList = class_copyMethodList(objectRuntimeClass, &count);
	    for (NSInteger i = 0; i < count; i++) {
	        Method method = methodList[i];
	        NSString *methodName = NSStringFromSelector(method_getName(method));
	        NSLog(@"method Name = %@\n", methodName);
	    }
	    free(methodList);
	    return @"";
	}

在另一个类中分别创建两个KVOObject对象，其中一个对象被观察者通过KVO的方式监听，另一个对象则始终没有被监听。在KVO前后分别打印两个对象的关键信息，看KVO前后有什么变化。

	@property (nonatomic, strong) KVOObject *object1;
	@property (nonatomic, strong) KVOObject *object2;
	
	self.object1 = [[KVOObject alloc] init];
	self.object2 = [[KVOObject alloc] init];
	[self.object1 description];
	[self.object2 description];
	
	[self.object1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
	[self.object1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
	
	[self.object1 description];
	[self.object2 description];
	
	self.object1.name = @"lxz";
	self.object1.age = 20;



我们发现对象被KVO后，其真正类型变为了NSKVONotifying_KVOObject类，已经不是之前的类了。KVO会在运行时动态创建一个新类，将对象的isa指向新创建的类，新类是原类的子类，命名规则是NSKVONotifying_xxx的格式。KVO为了使其更像之前的类，还会将对象的class实例方法重写，使其更像原类。

在上面的代码中还发现了_isKVOA方法，这个方法可以当做使用了KVO的一个标记，系统可能也是这么用的。如果我们想判断当前类是否是KVO动态生成的类，就可以从方法列表中搜索这个方法。
