## RAC 总结
### 1学习 RAC 我们首先要了解 RAC 都有哪些类

* RACSignal  信号类

* RACSubject  信号提供者,自己可以充当信号,又能发送信号.使用场景:通常用来代替代理,有了它,就不必要定义代理了。
RACSequence  信号的集合

* RACMulticastConnection 用于当一个信号,被多次订阅时,为了保证创建信号时,避免多次调用创建信号中的block,造成副作用,可以使用这个类处理。

* RACCommand  RAC中用于处理事件的类，可以把事件如何处理,事件中的数据如何传递，包装到这个类中，他可以很方便的监控事件的执行过程。

### RACSignal:

 RACSiganl:信号类,一般表示将来有数据传递，只要有数据改变，信号内部接收到数据，就会马上发出数据。

 信号类(RACSiganl)，只是表示当数据改变时，信号内部会发出数据，它本身不具备发送信号的能力，而是交给内部一个订阅者去发出。

 默认一个信号都是冷信号，也就是值改变了，也不会触发，只有订阅了这个信号，这个信号才会变为热信号，值改变了才会触发。

如何订阅信号：调用信号RACSignal的subscribeNext就能订阅。

//1.创建信号

	RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
		//3.发送信号
		[subscriber sendNext:@"发送信号"];
		/**
		如果不在发送数据，最好发送信号完成，内部会自动调用[RACDisposable disposable]取消订阅信号
		*/
		[subscriber sendCompleted];
		//取消订阅方法
		return [RACDisposable disposableWithBlock:^{
			//block调用时刻：当信号发送完成或者发送错误，就会自动执行这个block,取消订阅信号
			// 执行完Block后，当前信号就不在被订阅了。
			NSLog(@"信号销毁了");
		}];
	}];

//2.订阅信号

	[signal subscribeNext:^(id x) {
	NSLog(@"订阅信号:%@",x);
	}];

### RACSignal底层实现

     1.创建信号，首先把didSubscribe保存到信号中，还不会触发。

     2.当信号被订阅，也就是调用signal的subscribeNext:nextBlock

     2.1 subscribeNext内部会调用siganl的didSubscribe

     2.2 subscribeNext内部会创建订阅者subscriber，并且把nextBlock保存到subscriber中

     3.siganl的didSubscribe中调用[subscriber sendNext:@"发送信号"];

     3.1 sendNext底层其实就是执行subscriber的nextBlock

注意:

当一个信号被全局变量保存值的时候,我们要手动取消订阅

	RACSignal *signal1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	
	/**
	有一个全局变量保存值就不会走下面取消订阅方法
	*/
	_subscriber = subscriber;
	
	[subscriber sendNext:@"123"];
	[subscriber sendCompleted];
	
	return [RACDisposable disposableWithBlock:^{
	NSLog(@"走销毁这个方法了");
	}];
	}];

 

RACDisposable 取消订阅

//订阅信号返回RACDisposable

	RACDisposable *disposable = [signal1 subscribeNext:^(id x) {
	NSLog(@"接收打印的值:%@",x);
	}];

// 默认一个信号发送数据完毕们就会主动取消订阅.
// 只要订阅者在,就不会自动取消信号订阅
// 手动取消订阅者
[disposable dispose];

### RACSubscriber:

表示订阅者的意思，用于发送信号，这是一个协议，不是一个类，只要遵守这个协议，并且实现方法才能成为订阅者。通过create创建的信号，都有一个订阅者，帮助他发送数据。

RACDisposable:用于取消订阅或者清理资源，当信号发送完成或者发送错误的时候，就会自动触发它。

使用场景:不想监听某个信号时，可以通过它主动取消订阅信号。


#### RACSubject:

RACSubject:信号提供者，自己可以充当信号，又能发送信号。

使用场景:通常用来代替代理，有了它，就不必要定义代理了。

#### RACReplaySubject:

重复提供信号类，RACSubject的子类。

RACReplaySubject与RACSubject区别:

RACReplaySubject可以先发送信号，在订阅信号，RACSubject就不可以。

 使用场景一:如果一个信号每被订阅一次，就需要把之前的值重复发送一遍，使用重复提供信号类。

 使用场景二:可以设置capacity数量来限制缓存的value的数量,即只缓充最新的几个值。

 // RACSubject使用步骤

// 1.创建信号 [RACSubject subject]，跟RACSiganl不一样，创建信号时没有block。

// 2.订阅信号 - (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock

// 3.发送信号 sendNext:(id)value

	//1.创建信号
	RACSubject *subject = [RACSubject subject];
	//2.订阅信号
	[subject subscribeNext:^(id x) {
	//当信号发出新值，就会调用.
	NSLog(@"第一个订阅者%@",x);
	}];
	
	[subject subscribeNext:^(id x) {
	NSLog(@"第二个订阅者%@",x);
	}];
	
	//3.发送信号
	[subject sendNext:@"123456"];
	//4.发送信号完成,内部会自动取消订阅者
	[subject sendCompleted];
	
	//输出:
	// 第一个订阅者123456
	// 第二个订阅者123456

#### RACReplaySubject

RACReplaySubject使用步骤:

1.创建信号 [RACSubject subject]，跟RACSiganl不一样，创建信号时没有block。

2.可以先订阅信号，也可以先发送信号。

2.1 订阅信号 - (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock

2.2 发送信号 sendNext:(id)value 

RACReplaySubject:底层实现和RACSubject不一样。

1.调用sendNext发送信号，把值保存起来，然后遍历刚刚保存的所有订阅者，一个一个调用订阅者的nextBlock。

2.调用subscribeNext订阅信号，遍历保存的所有值，一个一个调用订阅者的nextBlock

	//1. RACReplaySubject *replaySubject = [RACReplaySubject subject];
	
	//2.发送信号 [replaySubject sendNext:@"1"];
	
	[replaySubject sendNext:@"2"];
	
	//3.订阅信号 [replaySubject subscribeNext:^(id x) {
	
	NSLog(@"第一个订阅者接收到的数据%@",x);
	
	}];
	
	[replaySubject subscribeNext:^(id x) {
	
	NSLog(@"第二个订阅者接收到的数据%@",x);
	
	}];
	
	//输出:
	
	//第一个订阅者接收到的数据
	
	1 //第一个订阅者接收到的数据
	
	2 //第二个订阅者接收到的数据
	
	1 //第二个订阅者接收到的数据2

 

### RAC集合

* RACTuple:元组类,类似NSArray,用来包装值.

/**
RACTuple:元组类,类似NSArray,用来包装值.
*/

	//元组
	RACTuple *tuple = [RACTuple tupleWithObjectsFromArray:@[@"123",@"345",@1]];
	NSString *first = tuple[0];
	NSLog(@"%@",first);
	
	//输出:
	//123

* 数组

		NSArray *arr = @[@"213",@"321",@1];
		
		//RAC 集合
		// RACSequence *sequence = arr.rac_sequence;
		// 把集合转换成信号
		
		// RACSignal *signal = sequence.signal;
		
		// 订阅集合信号,内部会自动遍历所有的元素发出来
		
		// [signal subscribeNext:^(id x) {
		
			// NSLog(@"遍历数组%@",x);
		// }];
		//输出:
		/**
		遍历数组213
		遍历数组321
		遍历数组1
		*/
		
		//高级写法
		[arr.rac_sequence.signal subscribeNext:^(id x) {
			NSLog(@"高级写法遍历数组打印%@",x);
		}];

		//输出:
		/**
		高级写法遍历数组打印213
		高级写法遍历数组打印321
		高级写法遍历数组打印1
		*/

* 字典

		NSDictionary *dict = @{@"sex":@"女",@"name":@"苍老师",@"age":@18};
		
		//转换成集合
		[dict.rac_sequence.signal subscribeNext:^(RACTuple *x) {
		// NSString *key = x[0];
		// NSString *value = x[1];
		// NSLog(@"%@ %@",key,value);
		// RACTupleUnpack:用来解析元组
		// 宏里面的参数,传需要解析出来的变量名
		//= 右边,放需要解析的元组
		RACTupleUnpack(NSString *key, NSString *value) = x;
		NSLog(@"%@ %@",key,value);
		
		}];
		//输出:
		/**
		sex 女
		name 苍老师
		age 18
		*/
		
		字典转模型
		NSString *filePath = [[NSBundle mainBundle] pathForResource:@"flags.plist" ofType:nil];
		NSArray *dictArr = [NSArray arrayWithContentsOfFile:filePath];
		
		// NSMutableArray *arr = [NSMutableArray array];
		// // rac_sequence注意点：调用subscribeNext，并不会马上执行nextBlock，而是会等一会。
		
		// [dictArr.rac_sequence.signal subscribeNext:^(NSDictionary *x) {
		// Flag *flag = [Flag flagWithDict:x];
		// [arr addObject:flag];
		// }];
		
		//高级用法
		// map:映射的意思，目的：把原始值value映射成一个新值
		// array: 把集合转换成数组
		// 底层实现：当信号被订阅，会遍历集合中的原始值，映射成新值，并且保存到新的数组里。
		NSArray *arr = [[dictArr.rac_sequence map:^id(NSDictionary *value) {
		return [Flag flagWithDict:value];
		}] array];
		
		NSLog(@"%@",arr);


* RACMulticastConnection:

RACMulticastConnection:用于当一个信号，被多次订阅时，为了保证创建信号时，避免多次调用创建信号中的block，造成副作用，可以使用这个类处理。

 使用注意:RACMulticastConnection通过RACSignal的-publish或者-muticast:方法创建.

RACMulticastConnection简单使用:

 RACMulticastConnection使用步骤:

 1.创建信号 + (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe

 2.创建连接 RACMulticastConnection *connect = [signal publish];

 3.订阅信号,注意：订阅的不在是之前的信号，而是连接的信号。 [connect.signal subscribeNext:nextBlock]

 4.连接 [connect connect]

    

 RACMulticastConnection底层原理:

 1.创建connect，connect.sourceSignal -> RACSignal(原始信号)  connect.signal -> RACSubject

 2.订阅connect.signal，会调用RACSubject的subscribeNext，创建订阅者，而且把订阅者保存起来，不会执行block。

 3.[connect connect]内部会订阅RACSignal(原始信号)，并且订阅者是RACSubject

 3.1.订阅原始信号，就会调用原始信号中的didSubscribe

 3.2 didSubscribe，拿到订阅者调用sendNext，其实是调用RACSubject的sendNext

 4.RACSubject的sendNext,会遍历RACSubject所有订阅者发送信号。

 4.1 因为刚刚第二步，都是在订阅RACSubject，因此会拿到第二步所有的订阅者，调用他们的nextBlock

 需求：假设在一个信号中发送请求，每次订阅一次都会发送请求，这样就会导致多次请求。

 解决：使用RACMulticastConnection就能解决.

	//1.创建信号
	RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	NSLog(@"发送请求");
	
	[subscriber sendNext:@"1"];
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	//2.订阅信号
	[signal subscribeNext:^(id x) {
	
	NSLog(@"接收数据");
	}];
	[signal subscribeNext:^(id x) {
	
	NSLog(@"接收数据");
	}];
	
	// 3.运行结果，会执行两遍发送请求，也就是每次订阅都会发送一次请求

	// RACMulticastConnection:解决重复请求问题
	// 1.创建信号
	RACSignal *connectionSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	NSLog(@"发送请求--");
	[subscriber sendNext:@1];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];

	//2.创建连接
	RACMulticastConnection *connect = [connectionSignal publish];
	
	//3.订阅信号
	// 注意：订阅信号，也不能激活信号，只是保存订阅者到数组，必须通过连接,当调用连接，就会一次性调用所有订阅者的sendNext:
	[connect.signal subscribeNext:^(id x) {
	NSLog(@"订阅者一信号--");
	}];
	[connect.signal subscribeNext:^(id x) {
	
	NSLog(@"订阅者二信号--");
	
	}];
	//4.连接,激活信号
	[connect connect];
	
	//输出:
	/**
	发送请求--
	订阅者一信号--
	订阅者二信号--
	*/

* RACCommand

RACCommand:RAC中用于处理事件的类，可以把事件如何处理,事件中的数据如何传递，包装到这个类中，他可以很方便的监控事件的执行过程。

  使用场景:监听按钮点击，网络请求

### 一、RACCommand使用步骤:

* 1.创建命令 initWithSignalBlock:(RACSignal * (^)(id input))signalBlock

* 2.在signalBlock中，创建RACSignal，并且作为signalBlock的返回值

* 3.执行命令 - (RACSignal *)execute:(id)input

### 二、RACCommand使用注意:

* 1.signalBlock必须要返回一个信号，不能传nil.

* 2.如果不想要传递信号，直接创建空的信号[RACSignal empty];

* 3.RACCommand中信号如果数据传递完，必须调用[subscriber sendCompleted]，这时命令才会执行完毕，否则永远处于执行中。

### 三、RACCommand设计思想：内部signalBlock为什么要返回一个信号，这个信号有什么用。

* 1.在RAC开发中，通常会把网络请求封装到RACCommand，直接执行某个RACCommand就能发送请求。

* 2.当RACCommand内部请求到数据的时候，需要把请求的数据传递给外界，这时候就需要通过signalBlock返回的信号传递了。

### 四、如何拿到RACCommand中返回信号发出的数据。

* 1.RACCommand有个执行信号源executionSignals，这个是signal of signals(信号的信号),意思是信号发出的数据是信号，不是普通的类型。

* 2.订阅executionSignals就能拿到RACCommand中返回的信号，然后订阅signalBlock返回的信号，就能获取发出的值。

### 五、监听当前命令是否正在执行executing

### 六、使用场景,监听按钮点击，网络请求

	//1.创建命令
	RACCommand *command = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
	
	NSLog(@"执行命令");
	//创建空信号,必须返回信号
	//return [RACSignal empty];
	
	//2.创建信号,用来传递数据
	return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	
	[subscriber sendNext:@"请求数据"];
	
	//注意:数据传递完，最好调用sendCompleted，这时命令才执行完毕。
	[subscriber sendCompleted];
	
	return [RACDisposable disposableWithBlock:^{
	NSLog(@"数据销毁");
	}];
	}];
	}];

	//强引用命令，不要被销毁，否则接收不到数据
	_command = command;
	
	//监听事件有咩有完成
	[command.executing subscribeNext:^(id x) {
	if ([x boolValue] == YES) { // 当前正在执行
	NSLog(@"当前正在执行");
	}else{
	// 执行完成/没有执行
	NSLog(@"执行完成/没有执行");
	}
	
	}];
	
	//执行命令
	[self.command execute:@1];

### RAC高级用法

	//创建信号中信号
	RACSubject *signalOfSignals = [RACSubject subject];
	RACSubject *signalA = [RACSubject subject];
	RACSubject *signalB = [RACSubject subject];
	// 订阅信号
	// [signalOfSignals subscribeNext:^(RACSignal *x) {
	// [x subscribeNext:^(id x) {
	// NSLog(@"%@",x);
	// }];
	// }];
	// switchToLatest:获取信号中信号发送的最新信号
	[signalOfSignals.switchToLatest subscribeNext:^(id x) {
	
	NSLog(@"%@",x);
	}];
	
	// 发送信号
	[signalOfSignals sendNext:signalA];
	
	[signalA sendNext:@1];
	[signalB sendNext:@"BB"];
	[signalA sendNext:@"11"];

RACCommand:处理事件

RACCommand:不能返回一个空的信号

	// 1.创建命令
	RACCommand *command = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
	// input:执行命令传入参数
	// Block调用:执行命令的时候就会调用
	NSLog(@"%@",input);
	
	return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	
	// 发送数据
	[subscriber sendNext:@"执行命令产生的数据"];
	
	return nil;
	}];
	}];

// 如何拿到执行命令中产生的数据
// 订阅命令内部的信号
// 1.方式一:直接订阅执行命令返回的信号
// 2.方式二:

// 2.执行命令
RACSignal *signal = [command execute:@1];

// 3.订阅信号
[signal subscribeNext:^(id x) {
NSLog(@"%@",x);
}];

 

### RAC 中常见的宏:

使用宏定义要单独导入 #import <RACEXTScope.h>


#### 一. RAC(TARGET, [KEYPATH, [NIL_VALUE]]):用于给某个对象的某个属性绑定

只要文本框文字改变，就会修改label的文字
RAC(self.labelView,text) = _textField.rac_textSignal;

#### 二.RACObserve(self, name):监听某个对象的某个属性,返回的是信号。

	[RACObserve(self.view, center) subscribeNext:^(id x) {
	NSLog(@"%@",x);
	}];

当RACObserve放在block里面使用时一定要加上weakify，不管里面有没有使用到self；否则会内存泄漏，因为RACObserve宏里面就有一个self

	@weakify(self);
	RACSignal *signal3 = [anotherSignal flattenMap:^(NSArrayController *arrayController) {
	Avoids a retain cycle because of RACObserve implicitly referencing self
	@strongify(self);
	return RACObserve(arrayController, items);
	}];

 

#### 三.@weakify(Obj)和@strongify(Obj)

一般两个都是配套使用,在主头文件(ReactiveCocoa.h)中并没有导入，需要自己手动导入，RACEXTScope.h才可以使用。但是每次导入都非常麻烦，只需要在主头文件自己导入就好了

#### 四.RACTuplePack：把数据包装成RACTuple（元组类）

把参数中的数据包装成元组
RACTuple *tuple = RACTuplePack(@10,@20);

####五.RACTupleUnpack：把RACTuple（元组类）解包成对应的数据

// 把参数中的数据包装成元组
RACTuple *tuple = RACTuplePack(@"xmg",@20);

// 解包元组，会把元组的值，按顺序给参数里面的变量赋值
// name = @"xmg" age = @20
RACTupleUnpack(NSString *name,NSNumber *age) = tuple;

 

## RAC常用用法:

### 1.监听按钮的点击事件:

	UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
	button.frame = CGRectMake(100, 100, 80, 40);
	[button setTitle:@"点击事件" forState:UIControlStateNormal];
	[button setBackgroundColor:[UIColor redColor]];
	[self.view addSubview:button];
	
	[[button rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
	NSLog(@"按钮被点击了");
	}];

### 2.代理的使用:

	@interface RedView : UIView
	 
	//*  */
	@property (nonatomic, strong) RACSubject *btnClickSignal;
	 
	@end
	 
	#import "RedView.h"
	 
	@implementation RedView
	- (void)awakeFromNib {
	 
	    [super awakeFromNib];
	}
	 
	//懒加载信号
	- (RACSubject *)btnClickSignal {
	 
	    if (_btnClickSignal == nil) {
	        _btnClickSignal = [RACSubject subject];
	    }
	    return _btnClickSignal;
	}
	 
	- (IBAction)btnClick:(UIButton *)sender {
	    
	    [self.btnClickSignal sendNext:@"按钮被点击了"];
	}
	 
	@end
	// 只要传值,就必须使用RACSubject
	RedView *redView = [[[NSBundle mainBundle] loadNibNamed:@"RedView" owner:nil options:nil] lastObject];
	redView.frame = CGRectMake(0, 140, self.view.bounds.size.width, 200);
	[self.view addSubview:redView];
	[redView.btnClickSignal subscribeNext:^(id x) {
	NSLog(@"---点击按钮了,触发了事件");
	}];
	
	// 把控制器调用didReceiveMemoryWarning转换成信号
	//rac_signalForSelector:监听某对象有没有调用某方法
	
	[[self rac_signalForSelector:@selector(didReceiveMemoryWarning)] subscribeNext:^(id x) {
	NSLog(@"控制器调用didReceiveMemoryWarning");
	
	}];

### 3.KVO

把监听redV的center属性改变转换成信号，只要值改变就会发送信号

	 // observer:可以传入nil
	    [[redView rac_valuesAndChangesForKeyPath:@"center" options:NSKeyValueObservingOptionNew observer:nil] subscribeNext:^(id x) {
	        NSLog(@"center:%@",x);
	    }];
	4.代替通知
	
	UITextField *textfield = [[UITextField alloc] initWithFrame:CGRectMake(100, 340, 150, 40)];
	textfield.placeholder = @"监听键盘的弹起";
	textfield.borderStyle = UITextBorderStyleRoundedRect;
	[self.view addSubview:textfield];
	
	[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIKeyboardWillShowNotification object:nil] subscribeNext:^(id x) {
	NSLog(@"监听键盘的高度:%@",x);
	}];

### 5.监听文字的改变


	[textfield.rac_textSignal subscribeNext:^(id x) {
	     
	     NSLog(@"输入框中文字改变了---%@",x);
	 }];
	 
### 6.处理多个请求，都返回结果的时候，统一做处理.

	RACSignal *request1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	
	// 发送请求1
	[subscriber sendNext:@"发送请求1"];
	return nil;
	}];
	
	RACSignal *request2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	// 发送请求2
	[subscriber sendNext:@"发送请求2"];
	return nil;
	}];
	
	// 使用注意：几个信号，参数一的方法就几个参数，每个参数对应信号发出的数据。
	[self rac_liftSelector:@selector(updateUIWithR1:r2:) withSignalsFromArray:@[request1,request2]];
	
	
	// 更新UI
	- (void)updateUIWithR1:(id)data r2:(id)data1
	{
	NSLog(@"更新UI%@ %@",data,data1);
	}

 

## ReactiveObjC常见操作方法介绍:

### 1.1 v操作须知

 所有的信号（RACSignal）都可以进行操作处理，因为所有操作方法都定义在RACStream.h中，而RACSignal继承RACStream。

### 1.2 ReactiveObjC操作思想

运用的是Hook（钩子）思想，Hook是一种用于改变API(应用程序编程接口：方法)执行结果的技术.

 Hook用处：截获API调用的技术。

 Hook原理：在每次调用一个API返回结果之前，先执行你自己的方法，改变结果的输出。

 RAC开发方式：RAC中核心开发方式，也是绑定，之前的开发方式是赋值，而用RAC开发，应该把重心放在绑定，也就是可以在创建一个对象的时候，就绑定好以后想要做的事情，而不是等赋值之后在去做事情。

 列如：把数据展示到控件上，之前都是重写控件的setModel方法，用RAC就可以在一开始创建控件的时候，就绑定好数据。

### 1.3 ReactiveObjC核心方法bind

 ReactiveObjC操作的核心方法是bind（绑定）,给RAC中的信号进行绑定，只要信号一发送数据，就能监听到，从而把发送数据改成自己想要的数据。

在开发中很少使用bind方法，bind属于RAC中的底层方法，RAC已经封装了很多好用的其他方法，底层都是调用bind，用法比bind简单.

 bind方法简单介绍和使用。

// 假设想监听文本框的内容，并且在每次输出结果的时候，都在文本框的内容拼接一段文字“输出：”

// 方式一:在返回结果后，拼接。
	
	[_textField.rac_textSignal subscribeNext:^(id x) {
	
	NSLog(@"输出:%@",x);
	
	}];

// 方式二:在返回结果前，拼接，使用RAC中bind方法做处理。

// bind方法参数:需要传入一个返回值是RACStreamBindBlock的block参数

// RACStreamBindBlock是一个block的类型，返回值是信号，参数（value,stop），因此参数的block返回值也是一个block。

// RACStreamBindBlock:

// 参数一(value):表示接收到信号的原始值，还没做处理

// 参数二(*stop):用来控制绑定Block，如果*stop = yes,那么就会结束绑定。

// 返回值：信号，做好处理，在通过这个信号返回出去，一般使用RACReturnSignal,需要手动导入头文件RACReturnSignal.h。

// bind方法使用步骤:

// 1.传入一个返回值RACStreamBindBlock的block。

// 2.描述一个RACStreamBindBlock类型的bindBlock作为block的返回值。

// 3.描述一个返回结果的信号，作为bindBlock的返回值。

// 注意：在bindBlock中做信号结果的处理。

// 底层实现:

// 1.源信号调用bind,会重新创建一个绑定信号。

// 2.当绑定信号被订阅，就会调用绑定信号中的didSubscribe，生成一个bindingBlock。

// 3.当源信号有内容发出，就会把内容传递到bindingBlock处理，调用bindingBlock(value,stop)

// 4.调用bindingBlock(value,stop)，会返回一个内容处理完成的信号（RACReturnSignal）。

// 5.订阅RACReturnSignal，就会拿到绑定信号的订阅者，把处理完成的信号内容发送出来。


// 注意:不同订阅者，保存不同的nextBlock，看源码的时候，一定要看清楚订阅者是哪个。

// 这里需要手动导入#import <ReactiveCocoa/RACReturnSignal.h>，才能使用
RACReturnSignal。

	[[_textField.rac_textSignal bind:^RACStreamBindBlock{
	
	// 什么时候调用:
	// block作用:表示绑定了一个信号.
	
	return ^RACStream *(id value, BOOL *stop){
	
	// 什么时候调用block:当信号有新的值发出，就会来到这个block。
	
	// block作用:做返回值的处理
	
	// 做好处理，通过信号返回出去.
	return [RACReturnSignal return:[NSString stringWithFormat:@"输出:%@",value]];
	};
	
	}] subscribeNext:^(id x) {
	
	NSLog(@"%@",x);
	
	}];

### flattenMap

     flattenMap使用步骤:

     1.传入一个block，block类型是返回值RACStream，参数value

     2.参数value就是源信号的内容，拿到源信号的内容做处理

     3.包装成RACReturnSignal信号，返回出去。

     

     flattenMap底层实现:

     0.flattenMap内部调用bind方法实现的,flattenMap中block的返回值，会作为bind中bindBlock的返回值。

     1.当订阅绑定信号，就会生成bindBlock。

     2.当源信号发送内容，就会调用bindBlock(value, *stop)

     3.调用bindBlock，内部就会调用flattenMap的block，flattenMap的block作用：就是把处理好的数据包装成信号。

     4.返回的信号最终会作为bindBlock中的返回信号，当做bindBlock的返回信号。

     5.订阅bindBlock的返回信号，就会拿到绑定信号的订阅者，把处理完成的信号内容发送出来。
     

	[[_textField.rac_textSignal flattenMap:^RACStream *(id value) {
	// block什么时候 : 源信号发出的时候，就会调用这个block。
	
	// block作用 : 改变源信号的内容。
	// 返回值：绑定信号的内容.
	return [RACReturnSignal return:[NSString stringWithFormat:@"输出:%@",value]];
	}] subscribeNext:^(id x) {
	// 订阅绑定信号，每当源信号发送内容，做完处理，就会调用这个block。
	NSLog(@"输出:flattenMap%@",x);
	}];

 

### map

     Map使用步骤:

     1.传入一个block,类型是返回对象，参数是value

     2.value就是源信号的内容，直接拿到源信号的内容做处理

     3.把处理好的内容，直接返回就好了，不用包装成信号，返回的值，就是映射的值。

 

     Map底层实现:

     0.Map底层其实是调用flatternMap,Map中block中的返回的值会作为flatternMap中block中的值。

     1.当订阅绑定信号，就会生成bindBlock。

     3.当源信号发送内容，就会调用bindBlock(value, *stop)

     4.调用bindBlock，内部就会调用flattenMap的block

     5.flattenMap的block内部会调用Map中的block，把Map中的block返回的内容包装成返回的信号。

     5.返回的信号最终会作为bindBlock中的返回信号，当做bindBlock的返回信号。

     6.订阅bindBlock的返回信号，就会拿到绑定信号的订阅者，把处理完成的信号内容发送出来。

	[[_textField.rac_textSignal map:^id(id value) {
	// 当源信号发出，就会调用这个block，修改源信号的内容
	// 返回值：就是处理完源信号的内容。
	return [NSString stringWithFormat:@"输出:%@",value];
	}] subscribeNext:^(id x) {
	
	NSLog(@"map输出%@",x);
	}];

 

### FlatternMap和Map的区别

 1.FlatternMap中的Block返回信号。

 2.Map中的Block返回对象。

 3.开发中，如果信号发出的值不是信号，映射一般使用Map

 4.开发中，如果信号发出的值是信号，映射一般使用FlatternMap。

 总结：signalOfsignals用FlatternMap。

	//创建信号中的信号
	RACSubject *signalOfsignals = [RACSubject subject];
	RACSubject *signal = [RACSubject subject];
	
	[[signalOfsignals flattenMap:^RACStream *(id value) {
	
	//当signalOfsignals的signals发出信号才会调用
	return value;
	}] subscribeNext:^(id x) {
	// 只有signalOfsignals的signal发出信号才会调用，因为内部订阅了bindBlock中返回的信号，也就是flattenMap返回的信号。
	// 也就是flattenMap返回的信号发出内容，才会调用。
	
	NSLog(@"%@aaa",x);
	}];
	
	// 信号的信号发送信号
	[signalOfsignals sendNext:signal];
	
	// 信号发送内容
	[signal sendNext:@1];

 

### RAC操作之组合

#### 1.concat

按一定顺序拼接信号，当多个信号发出的时候，有顺序的接收信号

	//一.concat:按一定顺序拼接信号，当多个信号发出的时候，有顺序的接收信号。
	RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@1];
	[subscriber sendCompleted];
	return nil;
	}];
	
	RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@2];
	[subscriber sendCompleted];
	
	return nil;
	}];

	// 把signalA拼接到signalB后，signalA发送完成，signalB才会被激活。
	RACSignal *concatSignal = [signalA concat:signalB];
	
	// 以后只需要面对拼接信号开发。
	// 订阅拼接的信号，不需要单独订阅signalA，signalB
	// 内部会自动订阅。
	// 注意：第一个信号必须发送完成，第二个信号才会被激活
	[concatSignal subscribeNext:^(id x) {
	
	NSLog(@"%@",x);
	}];

	// concat底层实现:
	// 1.当拼接信号被订阅，就会调用拼接信号的didSubscribe
	// 2.didSubscribe中，会先订阅第一个源信号（signalA）
	// 3.会执行第一个源信号（signalA）的didSubscribe
	// 4.第一个源信号（signalA）didSubscribe中发送值，就会调用第一个源信号（signalA）订阅者的nextBlock,通过拼接信号的订阅者把值发送出来.
	// 5.第一个源信号（signalA）didSubscribe中发送完成，就会调用第一个源信号（signalA）订阅者的completedBlock,订阅第二个源信号（signalB）这时候才激活（signalB）。
	// 6.订阅第二个源信号（signalB）,执行第二个源信号（signalB）的didSubscribe
	// 7.第二个源信号（signalA）didSubscribe中发送值,就会通过拼接信号的订阅者把值发送出来.

 

#### 2.then

用于连接两个信号，当第一个信号完成，才会连接then返回的信号

then:用于连接两个信号，当第一个信号完成，才会连接then返回的信号

注意使用then，之前信号的值会被忽略掉.

底层实现：1、先过滤掉之前的信号发出的值。2.使用concat连接then返回的信号
	
	[[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	
	[subscriber sendNext:@1];
	[subscriber sendCompleted];
	
	return nil;
	
	}] then:^RACSignal *{
	return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@2];
	return nil;
	}];
	}] subscribeNext:^(id x) {
	// 只能接收到第二个信号的值，也就是then返回信号的值
	NSLog(@"%@",x);
	}];

 

### 3 merge

把多个信号合并为一个信号,任何一个信号有新值的时候就会调用.

merge: 把多个信号合并成一个信号

创建多个信号

	RACSignal *signalC = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	
	[subscriber sendNext:@3];
	
	return [RACDisposable disposableWithBlock:^{
	NSLog(@"信号释放了");
	}];
	}];
	
	RACSignal *signalD = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@4];
	
	return [RACDisposable disposableWithBlock:^{
	NSLog(@"信号释放了");
	}];
	}];

//合并信号.任何一个信号发送数据,都能监听到

RACSignal *mergeSignal = [signalC merge:signalD];
[mergeSignal subscribeNext:^(id x) {
NSLog(@"合并信号,任何一个数据发送数据,都能监听到%@",x);
}];

// 底层实现：
// 1.合并信号被订阅的时候，就会遍历所有信号，并且发出这些信号。
// 2.每发出一个信号，这个信号就会被订阅
// 3.也就是合并信号一被订阅，就会订阅里面所有的信号。
// 4.只要有一个信号被发出就会被监听。

 

### 4.zipWith

把两个信号压缩成一个信号，只有当两个信号同时发出信号内容时，并且把两个信号的内容合并成一个元组，才会触发压缩流的next事件。

	RACSignal *signalE = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@5];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	
	RACSignal *signalF = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@6];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	
	// 压缩信号A，信号B
	
	RACSignal *zipSignal = [signalE zipWith:signalF];
	
	[zipSignal subscribeNext:^(id x) {
	NSLog(@"压缩的信号%@",x);
	}];
	
	// 底层实现:
	// 1.定义压缩信号，内部就会自动订阅signalA，signalB
	// 2.每当signalA或者signalB发出信号，就会判断signalA，signalB有没有发出个信号，有就会把最近发出的信号都包装成元组发出。

 

### 5.combineLatest

将多个信号合并起来，并且拿到各个信号的最新的值,必须每个合并的signal至少都有过一次sendNext，才会触发合并的信号.

	RACSignal *signalG = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@5];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	
	RACSignal *signalH = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@6];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	
	// 把两个信号组合成一个信号,跟zip一样，没什么区别
	RACSignal *combineSignal = [signalG combineLatestWith:signalH];
	
	[combineSignal subscribeNext:^(id x) {
	NSLog(@"把两个信号组合成一个信号%@",x);
	}];
	
	// 底层实现：
	// 1.当组合信号被订阅，内部会自动订阅signalA，signalB,必须两个信号都发出内容，才会被触发。
	// 2.并且把两个信号组合成元组发出。

 

### 6.reduce聚合

用于信号发出的内容是元组，把信号发出元组的值聚合成一个值

	RACSignal *signalL = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@5];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	
	RACSignal *signalM = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@6];
	
	return [RACDisposable disposableWithBlock:^{
	
	}];
	}];
	
	RACSignal *reduceSignal = [RACSignal combineLatest:@[signalL,signalM] reduce:^id(NSNumber *num1, NSNumber *num2){
	return [NSString stringWithFormat:@"reduce聚合%@ %@",num1,num2];
	}];
	
	[reduceSignal subscribeNext:^(id x) {
	NSLog(@"%@",x);
	
	}];
	
	// // 底层实现:
	// 1.订阅聚合信号，每次有内容发出，就会执行reduceblcok，把信号内容转换成reduceblcok返回的值。

##ReactiveCocoa常见操作方法介绍:

### filter

### ignore

### ignoreValues

### takeUntilBlock

### distinctUntilChanged

### take

### takeLast

### takeUntil

### skip

### switchToLatest

###filter: 过滤信号，使用它可以获取满足条件的信号.


	//filter 过滤
	    //每次信号发出,会先执行过滤条件判断
	    [_textField.rac_textSignal filter:^BOOL(NSString *value) {
	       
	        return value.length > 3;
	    }];
###ignore:忽略完某些值的信号.

	// 内部调用filter过滤，忽略掉ignore的值
	    [[_textField.rac_textSignal ignore:@"1"] subscribeNext:^(id x) {
	        NSLog(@"ignore%@",x);
	    }];
	    
### ignoreValues 这个比较极端，忽略所有值，只关心Signal结束，也就是只取Comletion和Error两个消息，中间所有值都丢弃

	RACSignal *signal=[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@"1"];
	[subscriber sendNext:@"3"];
	[subscriber sendNext:@"15"];
	[subscriber sendNext:@"wujy"];
	[subscriber sendCompleted];
	return [RACDisposable disposableWithBlock:^{
	NSLog(@"执行清理");
	}];
	}];


	[[signal ignoreValues] subscribeNext:^(id x) {
		//它是没机会执行 因为ignoreValues已经忽略所有的next值
		NSLog(@"ignoreValues当前值：%@",x);
	} error:^(NSError *error) {
		NSLog(@"ignoreValues error");
	} completed:^{
		NSLog(@"ignoreValues completed");
	}];

 

### distinctUntilChanged:当上一次的值和当前的值有明显的变化就会发出信号，否则会被忽略掉。
	// 过滤，当上一次和当前的值不一样，就会发出内容。
	// 在开发中，刷新UI经常使用，只有两次数据不一样才需要刷新
	
	[[_textField.rac_textSignal distinctUntilChanged] subscribeNext:^(id x) {
	NSLog(@"distinctUntilChanged%@",x);
	}];
	
	take:从开始一共取N次的信号
	
	//1.创建信号
	RACSubject *signal = [RACSubject subject];
	//2.处理信号,订阅信号
	[[signal take:1] subscribeNext:^(id x) {
	NSLog(@"take:%@",x);
	
	}];
	
	// 3.发送信号
	[signal sendNext:@1];
	
	[signal sendNext:@2];
	
 

### takeLast:取最后N次的信号,前提条件，订阅者必须调用完成，因为只有完成，就知道总共有多少信号.

	//1.创建信号
	RACSubject *signal = [RACSubject subject];
	//2.处理信号,订阅信号
	[[signal takeLast:2] subscribeNext:^(id x) {
	NSLog(@"%@",x);
	
	}];
	
	//3.发送信号
	[signal sendNext:@1];
	[signal sendNext:@332];
	[signal sendNext:@333];
	[signal sendCompleted];

 

### takeUntil:(RACSignal *):获取信号直到执行完这个信号
	// 监听文本框的改变，知道当前对象被销毁
	[_textField.rac_textSignal takeUntil:self.rac_willDeallocSignal];
	
	takeUntilBlock 对于每个next值，运行block，当block返回YES时停止取值
	
	RACSignal *signal=[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@"1"];
	[subscriber sendNext:@"3"];
	[subscriber sendNext:@"15"];
	[subscriber sendNext:@"wujy"];
	[subscriber sendCompleted];
	return [RACDisposable disposableWithBlock:^{
	NSLog(@"执行清理");
	}];
	}];
	
	[[signal takeUntilBlock:^BOOL(NSString *x) {
	if ([x isEqualToString:@"15"]) {
	return YES;
	}
	return NO;
	}] subscribeNext:^(id x) {
	NSLog(@"takeUntilBlock 获取的值：%@",x);
	}];
	
	// 输出
	// takeUntilBlock 获取的值：1
	// takeUntilBlock 获取的值：3
	
 

### skip:(NSUInteger):跳过几个信号,不接受。

	 [[_textField.rac_textSignal skip:1] subscribeNext:^(id x) {
	        NSLog(@"跳过几个信号不接收%@",x);
	    }];
	switchToLatest:用于signalOfSignals（信号的信号），有时候信号也会发出信号，会在signalOfSignals中，获取signalOfSignals发送的最新信号。
	
	RACSubject *signalOfSignals = [RACSubject subject];
	RACSubject *signal = [RACSubject subject];
	[signalOfSignals sendNext:signal];
	// 获取信号中信号最近发出信号，订阅最近发出的信号。
	// 注意switchToLatest：只能用于信号中的信号
	
	[signalOfSignals.switchToLatest subscribeNext:^(id x) {
	
	NSLog(@"获取信号中信号最近发出信号，订阅最近发出的信号%@",x);
	}];

 

## RAC操作方法三.

### doNext
### deliverOn
### timeout
### interval
### delay
### retry
### replay
### throttle

//ReactiveCocoa操作方法之秩序。

	- (void)doNext {
	
	//doNext: 执行Next之前，会先执行这个Block
	//doCompleted: 执行sendCompleted之前，会先执行这个Block
	[[[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@1];
	[subscriber sendCompleted];
	return nil;
	}] doNext:^(id x) {
	// 执行[subscriber sendNext:@1];之前会调用这个Block
	NSLog(@"doNext");
	}] doCompleted:^{
	// 执行[subscriber sendCompleted];之前会调用这个Block
	NSLog(@"doCompleted");
	}] subscribeNext:^(id x) {
	NSLog(@"%@",x);
	
	}];
	}

 

//ReactiveCocoa操作方法之线程。

- (void)deliverOn {
    //deliverOn: 内容传递切换到制定线程中，副作用在原来线程中,把在创建信号时block中的代码称之为副作用。
    
    //subscribeOn: 内容传递和副作用都会切换到制定线程中。
}
//ReactiveCocoa操作方法之时间。

- (void)timeout {
//timeout：超时，可以让一个信号在一定的时间后，自动报错。
RACSignal *signal = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
return nil;
}] timeout:1 onScheduler:[RACScheduler currentScheduler]];

[signal subscribeNext:^(id x) {

NSLog(@"%@",x);

} error:^(NSError *error) {
// 1秒后会自动调用
NSLog(@"%@",error);
}];
}

 

//interval 定时：每隔一段时间发出信号

- (void)interval {

[[RACSignal interval:1 onScheduler:[RACScheduler currentScheduler]] subscribeNext:^(id x) {
NSLog(@"%@",x);
} error:^(NSError *error) {
NSLog(@"%@",error);
}];
}

 

//delay 延迟发送next

- (void)delay {

[[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

[subscriber sendNext:@1];
return nil;
}] delay:2] subscribeNext:^(id x) {
NSLog(@"%@",x);
}];
}

 

// ReactiveObjC操作方法之重复。

- (void)retry {
//retry重试 ：只要失败，就会重新执行创建信号中的block,直到成功.
__block int i = 0;
[[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {

if (i == 10) {
[subscriber sendNext:@1];
}else{
NSLog(@"接收到错误");
[subscriber sendError:nil];
}
i++;
return nil;

}] retry] subscribeNext:^(id x) {

NSLog(@"%@",x);

} error:^(NSError *error) {


}];
}

 

//replay重放：当一个信号被多次订阅,反复播放内容

- (void)replay {
RACSignal *signal = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {


[subscriber sendNext:@1];
[subscriber sendNext:@2];

return nil;
}] replay];

[signal subscribeNext:^(id x) {

NSLog(@"第一个订阅者%@",x);

}];

[signal subscribeNext:^(id x) {

NSLog(@"第二个订阅者%@",x);

}];
}

 

throttle

- (void)throttle {
RACSubject *signal = [RACSubject subject];
_signal = signal;
// 节流，在一定时间（1秒）内，不接收任何信号内容，过了这个时间（1秒）获取最后发送的信号内容发出。
[[signal throttle:1] subscribeNext:^(id x) {
NSLog(@"%@",x);
}];
}

 

[demo地址:](https://github.com/13662049573/TFY_ReactiveCocoaDemo.git)