ReactiveCocoa-Study
===================

学习ReactiveCocoa的总结


#FRP概念

#ReactiveCocoa框架

ReactiveCocoa（简称RAC）是基于FRP（functional reactive programming）设计的，RAC提供了一些信号类来捕获现在和未来的值。RACSignal。

先来看一下[leezhong再博文中提到的比喻](http://limboy.me/ios/2013/06/19/frp-reactivecocoa.html)，让你对有个ReactiveCocoa很好的理解：

>可以把信号想象成水龙头，只不过里面不是水，而是玻璃球(value)，直径跟水管的内径一样，这样就能保证玻璃球是依次排列，不会出现并排的情况(数据都是线性处理的，不会出现并发情况)。水龙头的开关默认是关的，除非有了接收方(subscriber)，才会打开。这样只要有新的玻璃球进来，就会自动传送给接收方。可以在水龙头上加一个过滤嘴(filter)，不符合的不让通过，也可以加一个改动装置，把球改变成符合自己的需求(map)。也可以把多个水龙头合并成一个新的水龙头(combineLatest:reduce:)，这样只要其中的一个水龙头有玻璃球出来，这个新合并的水龙头就会得到这个球。

信号可以是链式的、可以合并、可以捕获值得变化，它使软件更容易书写，不用去注册很多观察者来来更新某些模型或者界面的值。例如：当一个模型的值发生改变时，我们需要改变界面的显示，通常我们会用KVO的方式去改变，这样会产生很多代码，我们可以创建一个信号来监听模型的值，并把这个信号产生的新值赋给视图显示，这样只需要一句代码就实现了。

信号也可以用来表示异步线程，比如网络请求结束的信号，处理数据完成的信号。

RAC的主题就是信号，可以用信号来处理异步行为，包括代理、回调block、target-action事件机制、通知、KVO。

##Getting Started

由于ReaactiveCocoa比较难理解，开始使用之前，我们先介绍几个主要的概念。

###Streams
流，对应RACStream抽象类，他就像水管一样，所有的信号“玻璃球”都在里面传递。

信号有可能立即传进来或者是在某个时候，但是它是有顺序的，不能在第一个“玻璃球”没处理的情况下获取到它之后的一个玻璃球。

流对象一般我们不自己创建，那是非常糟糕的，更多的情况是通过signals和sequences来自动生成。

###Signals

信号，对应RACSignal类，在Stream里传输。

Signals一般表现为在将来会产生的数据，然后信号开始传递，data将会被接收，数据会包含在信号中，直到信号传递给订阅者。我们一定要为一个信号设置订阅者（subscribe）用来接收处理数据。

信号有三种不同的类型：

- ***NEXT*** 有新值产生，此类型的信号，Stream将会被继续传递，不像Cocoa collections，它结束于一个包含nil值得signal。
- ***Error*** 在信号结束之前发生了错误，这个类型信号包含了一个NSError对象，error必须易于理解，因为信号不包含在stream中传递的值。
- ***Completed*** 此信号是流完成传递工作的信号，表明流不会有新的值加入了。completed必须明确，信号中不会包含stream的其它值。

在steam的一个生命周期类，可以有很多个NEXT类型的信号，但是Error和Completed信号只有一个。

###Subscription

订阅者（subscriber），订阅者可以是任何对象，只要实现了RACSubscriber协议都可以是订阅者，它就像是在水龙头处等待“”玻璃球”信号流出的人。

订阅（subscription），订阅者可以订阅接收一个流的信号，通过[-subscribeNex:error:completed:]()方法，或者一些其它类似的更简便的方法。通常RACStream和RACSignal都会创建订阅，并给与它们实现的细节。

订阅retain它们的signal，依赖于信号的complete或error，当然也可以手动调用。

###Subjects

主题（Subject），对应RACSubject类，它是可以手动控制的信号。

你可以把它想象为一种变异的信号，就像NSMutableArray对于NSArray一样，它非常有用，是正常代码和RAC代码之间的桥梁，可以把正常对象的改变转变为RAC信号。

比如你有一个block，这个block需要进行一些异步处理，我们可以在处理完成时给一个Subject发送一些事件，返回这个Subject，这个Subject信号在处理完成就会被传递。

subjects也提出额外的行为，特别地，RACReplaySubject通常用于为将来的订阅者缓冲数据，就像需要去进行网络请求准备一些数据，完成后发送一个处理信号处理数据。

###Commands

命令（command），对应RACCommand类，它可创建订阅一个信号，通过一些动作（点击按钮），这使得与用户交互的app更容易工作。

动作通常是UI的事件，比如按钮的点击。commands在信号中是自动关闭的，这个关闭状态会在UI事件被触发的情况下开启，形成一个command。

在RAC中，给Button对象自动添加了rac_command属性。

###Connections

连接（connection），对应RACMuticastConnection类，它是一个订阅，包含了很多的订阅者。

信号默认是不活跃的，只有在一个新的订阅者添加后信号才会被传递处理，这个行为通常是有意义的，因为数据可能发生了某些变化需要订阅者处理，但是它也有一些问题，比如：在这个信号有负面影响的时候（网络请求，会消耗昂贵的资源），不能进行有效的处理。

connection通过RACSignal方法-publish或者-multicast:创建，必须确保有一个订阅信息被创建，无论这个connection被订阅了多少次；一个连接，其中一个信号被触发，这个隐私的订阅信息将被保留，直到这个连接所有的订阅信息被处理。

###Sequences

序列（Sequences），对应RACSequence类，是流的一个驱动器。

Sequences就像是一个集合，比如NSArray。但不是一个数组，

### Disposables

对应RACDisposable类，通常用来取消信号传递和清空数据。

它更多用来取消订阅信号，当一个订阅信息被取消，则订阅者不会收到任何信号。

###Schedulers

日程，对应RACScheduler类，定义连续队列，队列里存放信号，信号可以连续的传递值。

Schedulers是小型的GCD队列，但schedulers支持取消和顺序执行。除了+immediateScheduler，schedulers没有提供同步的操作。鼓励大家用signal替换掉block的工作。

##Basic Operators基本操作

###Performing side effects with signals给信号绑定另外的作用

信号在没有订阅之前是没有任何作用的，订阅之后，信号才可以有另外的作用，比如打印信息到控制台、做一个网络请求、更新用户界面。

另外的作用可以注入到信号中，他们不用立即响应，但是会在订阅之后产生它们的作用。

####Subscription

订阅方法将当前和将来的值放进一个信号中：

	RACSignal *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence.signal;
	// Outputs: A B C D E F G H I
	[letters subscribeNext:^(NSString *x) 
	{
   		NSLog(@"%@", x);
	}];
		
对与一个冷的信号，另外的动作将会在一个订阅发生后运行。

	__block unsigned subscriptions = 0;

	RACSignal *loggingSignal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
	    subscriptions++;
	    [subscriber sendCompleted];
	    return nil;
	}];
	
	// Outputs:
	// subscription 1
	[loggingSignal subscribeCompleted:^{
	    NSLog(@"subscription %u", subscriptions);
	}];
	
	// Outputs:
	// subscription 2
	[loggingSignal subscribeCompleted:^{
	    NSLog(@"subscription %u", subscriptions);
	}];
	
上面这个代码可以用connection来实现。

####在信号中注入动作

当一个信号已经被创建了，我们在信号执行时添加一些动作。

	__block unsigned subscriptions = 0;
	
	RACSignal *loggingSignal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
	    subscriptions++;
	    [subscriber sendCompleted];
	    return nil;
	}];
	
	// Does not output anything yet
	loggingSignal = [loggingSignal doCompleted:^{
	    NSLog(@"about to complete subscription %u", subscriptions);
	}];
	
	// Outputs:
	// about to complete subscription 1
	// subscription 1
	[loggingSignal subscribeCompleted:^{
	    NSLog(@"subscription %u", subscriptions);
	}];
	
###转换流数据

将一个流转换成一个新的流

####Mapping

-map：方法可以将流里的值进行一些变换，产生新值。

	RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence;
	
	// Contains: AA BB CC DD EE FF GG HH II
	RACSequence *mapped = [letters map:^(NSString *value) {
	    return [value stringByAppendingString:value];
	}];
	
####Filtering

-filter方法可以过滤掉一些信号，将一组信号队列过滤成一个新的队列。

	RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;
	
	// Contains: 2 4 6 8
	RACSequence *filtered = [numbers filter:^ BOOL (NSString *value) {
	    return (value.intValue % 2) == 0;
	}];
	
###合并流

####Concatenating 拼接

方法 -concat: 可将一串流拼接到另一串后面

	RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence;
	RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;
	
	// Contains: A B C D E F G H I 1 2 3 4 5 6 7 8 9
	RACSequence *concatenated = [letters concat:numbers];
	
####Flatening 合到一起

可以将两个流直接合到一起

如果是Sequences类型的流这样操作 concatenated:

	RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence;
	RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;
	RACSequence *sequenceOfSequences = @[ letters, numbers ].rac_sequence;
	
	// Contains: A B C D E F G H I 1 2 3 4 5 6 7 8 9
	RACSequence *flattened = [sequenceOfSequences flatten];
	
如果是Signal类型的流这样操作 merged:

	RACSubject *letters = [RACSubject subject];
	RACSubject *numbers = [RACSubject subject];
	RACSignal *signalOfSignals = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
	    [subscriber sendNext:letters];
	    [subscriber sendNext:numbers];
	    [subscriber sendCompleted];
	    return nil;
	}];
	
	RACSignal *flattened = [signalOfSignals flatten];
	
	// Outputs: A 1 B C 2
	[flattened subscribeNext:^(NSString *x) {
	    NSLog(@"%@", x);
	}];
	
	[letters sendNext:@"A"];
	[numbers sendNext:@"1"];
	[letters sendNext:@"B"];
	[letters sendNext:@"C"];
	[numbers sendNext:@"2"];

####Mapping and flattening

-flattenMap: 可以处理序列和信号

This can be used to extend or edit sequences:

	RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence;
	
	// Contains: 1 1 2 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9
	RACSequence *extended = [numbers flattenMap:^(NSString *num) {
	    return @[ num, num ].rac_sequence;
	}];
	
	// Contains: 1_ 3_ 5_ 7_ 9_
	RACSequence *edited = [numbers flattenMap:^(NSString *num) {
	    if (num.intValue % 2 == 0) {
	        return [RACSequence empty];
	    } else {
	        NSString *newNum = [num stringByAppendingString:@"_"];
	        return [RACSequence return:newNum]; 
	    }
	}];
	
Or create multiple signals of work which are automatically recombined:
	
	RACSignal *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence.signal;
	
	[[letters
	    flattenMap:^(NSString *letter) {
	        return [database saveEntriesForLetter:letter];
	    }]
	    subscribeCompleted:^{
	        NSLog(@"All database entries saved successfully.");
	    }];
	    
###Combining signals 合并信号

合并多个不同的信号成一个新的信号

####Sequencing 序列

-then: 开始与一个源信号，中途转换为一个新的信号。

我理解为一个信号的触发是另一个信号触发的前提，当第一个信号触发后，第二个信号就会被触发开始。

	RACSignal *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" "].rac_sequence.signal;
	
	// The new signal only contains: 1 2 3 4 5 6 7 8 9
	//
	// But when subscribed to, it also outputs: A B C D E F G H I
	RACSignal *sequenced = [[letters
	    doNext:^(NSString *letter) {
	        NSLog(@"%@", letter);
	    }]
	    then:^{
	        return [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" "].rac_sequence.signal;
	    }];
当letters触发时，sequenced信号就会被触发，但是sequenced信号所传递的数据是自己产生的数据。

这个经常会用在一个信号的额外动作，开始与另一个信号，但传递的数据是第二个信号自己的。

####Merging 合并

+merge: 方法将来自不同信号的值，都合并成一个信号的值。

	RACSubject *letters = [RACSubject subject];
	RACSubject *numbers = [RACSubject subject];
	RACSignal *merged = [RACSignal merge:@[ letters, numbers ]];
	
	// Outputs: A 1 B C 2
	[merged subscribeNext:^(NSString *x) {
	    NSLog(@"%@", x);
	}];
	
	[letters sendNext:@"A"];
	[numbers sendNext:@"1"];
	[letters sendNext:@"B"];
	[letters sendNext:@"C"];
	[numbers sendNext:@"2"];
	
传递给letters和numbers的值都会通过merged信号传递。

####Combining latest values

+combineLatest: 和+combinLatest:reduce: 方法会检测多个信号值得改变，当这些值得改变满足条件时，会产生一个合并后的信号。

	RACSubject *letters = [RACSubject subject];
	RACSubject *numbers = [RACSubject subject];
	RACSignal *combined = [RACSignal
	    combineLatest:@[ letters, numbers ]
	    reduce:^(NSString *letter, NSString *number) {
	        return [letter stringByAppendingString:number];
	    }];
	
	// Outputs: B1 B2 C2 C3
	[combined subscribeNext:^(id x) {
	    NSLog(@"%@", x);
	}];
	
	[letters sendNext:@"A"];
	[letters sendNext:@"B"];
	[numbers sendNext:@"1"];
	[numbers sendNext:@"2"];
	[letters sendNext:@"C"];
	[numbers sendNext:@"3"];
值得注意的是，这个信号只会在所有的输入都至少有一次有信号传输时，上面的例子中，@“A”没有打印，是因为numbers从没有被触发过。

####Switching

-switchToLatest 方法一般用在转发信号的时候，你可以转发信号1传递的值，也可以转发信号2传递的值。此方法可以在这之前随意切换。

	RACSubject *letters = [RACSubject subject];
	RACSubject *numbers = [RACSubject subject];
	RACSubject *signalOfSignals = [RACSubject subject];
	
	RACSignal *switched = [signalOfSignals switchToLatest];
	
	// Outputs: A B 1 D
	[switched subscribeNext:^(NSString *x) {
	    NSLog(@"%@", x);
	}];
	
	[signalOfSignals sendNext:letters];
	[letters sendNext:@"A"];
	[letters sendNext:@"B"];
	
	[signalOfSignals sendNext:numbers];
	[letters sendNext:@"C"];
	[numbers sendNext:@"1"];
	
	[signalOfSignals sendNext:letters];
	[numbers sendNext:@"2"];
	[letters sendNext:@"D"];