## objective-c内存管理准则
===
### 基本准则
1. 当你拥有一个对象时，你必须负责释放它。
2. 拥有是指通过alloc得到一个新的对象，或者对一个已有对象调用retain或者copy。
3. 释放可以通过调用release，也可以通过使用autorelease来延后释放。
4. 一个对象可能被拥有多次，只有当所有的拥有者都执行了释放动作后，这个对象才会被真正销毁。
5. 如果一个方法想要返回一个新的对象给调用者，autorelease是必须的方法。
6. 如果你使用alloc/retain/copy之外的方法得到一个对象，你只被保证在这一个调用周期内有效使用。
如果你使用基于事件回调机制的框架，比如cocoa，而且你需要长期使用这个对象，你必须对它执行retain或者copy。
7. 除此之外，你不需要也不能释放你正在使用的对象。否则会引起内存错误。

### 内存管理细节

#### obj-c使用基于引用计数的内存管理方式。
#### 当一个对象被alloc和初始化(init,initWithXXX)时，引用技术为1。
    NSString *s1 = [[NSString alloc] initWithFormat:@"%d",10]; // s1指向一个引用计数为1的新对象。s1负责释放。

#### 对一个对象执行简单赋值assign，没有改变引用计数。
    NSString *s1 = [[NSString alloc] initWithFormat:@"%d",10]; // s1指向一个引用计数为1的新对象。s1负责释放。
    NSString *s2 = s1; // s2和s1都指向同一个对象。对象的引用计数仍然为1。s2不负责释放对象。
    
#### 对一个对象执行retain，引用计数加1。返回的是同一个对象。
    NSString *s1 = [[NSString alloc] initWithFormat:@"%d",10]; // s1指向一个引用计数为1的新对象。s1负责释放。
    NSString *s2 = [s1 retain]; // 现在，s1和s2指向同一个对象，他们的引用计数是2。s1和s2都需负责释放一次。
    
#### 如果对象支持NSCopying协议，对它执行copy，返回一个新的引用计数为1的对象。原对象引用计数保持不变。
    NSString *s1 = [[NSString alloc] initWithFormat:@"%d",10]; // s1指向一个引用计数为1的新对象。s1负责释放。
    NSString *s2 = [s1 copy]; // s2指向一个引用计数为1的新对象。s2负责释放。

#### 对一个对象执行release，其背后的逻辑是先把这个对象的引用计数减1。然后判断是否是0，如果是0，则销毁这个对象。
    NSString *s1 = [[NSString alloc] initWithFormat:@"%d",10]; // s1指向一个引用计数为1的新对象。s1负责释放。
    [s1 release]; // s1的引用计数减1后变为0，对象被销毁。
    
    NSString *s2 = [[NSString alloc] initWithFormat:@"%d",10]; // s2指向一个引用计数为1的新对象。s2负责释放。
    NSString *s3 = [s2 retain]; // 现在，s3和s2指向同一个对象，他们的引用计数是2。s3也要负责释放一次。
    NSString *s4 = s3; // s4和s3,s2都指向同一个对象。但是这一步没有增加引用计数，对象的引用计数仍然为2。
    [s2 release]; // s2指向的对象的引用计数减1后变为1，对象仍然存在。
    [s3 release]; // s3指向的对象引用计数变为0，被销毁
    NSLog(@"%@", s4); // 内存错误，s4指向的对象已经被销毁。

#### autorelease延后释放
autorelease函数的逻辑是把对象注册到一个内存池对象中，当这个内存池对象执行清理动作时，
它会对所有在它这里注册的对象执行release。使用autorelease使你无需记得要在过程结束时释放你负责的对象，
特别是当你的函数有很多个出口时，这样做能减轻你的工作量。

    // s1指向的对象被创造出来，引用计数时1。然后注册到内存管理池中。s1无需再手工执行release。
    NSString *s1 = [[[NSString alloc] initWithFormat:@"%d",10] autorelease];
    NSString *s2 = [s1 retain]; //s2仍然需要负责手工release
    [s2 release];
    
    NSString *S3 = [[s1 retain] autorelease]; // 虽然怪异，但仍然是合法且有效的。
另外在函数返回一个新对象时，如果不想引起拥有权的转移，从而会引起不小心的bug，autorelease是必须的方法。
假设我们实现一个从整数返回字符串的函数:
```
- (NSString *)getStringFromInt:(int)x
{
    NSString *s = [[[NSString alloc] initWithFormat:@"%d", x] autorelease];
    return s;
}

- (void)test
{
    NSString *s1 = [self getStringFromInt:10]; // s1指向一个对象，但是它不拥有这个对象，所以无需释放
}
```

### 对象属性内存管理
一个属性可以声明为retain, copy, assign三种赋值方式中的其中一种。
另外属性可以指定为线程安全(atomic)或者非线程安全(nonatomic)。
假设现在有两个类，团队（Team）和领导Leader。团队除了领导之外，至少还有一个名称。我们再假设一个leader只领导一个team。
先看代码，然后来解释其中的蹊跷。
Leader.h
```
#import <Foundation/Foundation.h>

@class Team;

@interface Leader : NSObject

@property (nonatomic, assign) Team *team;

- (void)go;
@end
```

Leader.m
```
#import "Leader.h"
#import "Team.h"
@implementation Leader

- (id)init
{
    NSLog(@"Leader init");
    self = [super init];
    if (self) {
        
    }
    return self;
}

- (void)dealloc
{
    NSLog(@"Leader dealloc");
    [super dealloc];
}

- (void)go
{
    NSLog(@"Leader go");
}

@end
```

Team.h
```

#import <Foundation/Foundation.h>

@class Leader;

@interface Team : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, retain) Leader *leader;

- (void)go;
- (void)changeLeader;

@end
```

Team.m
```
#import "Team.h"
#import "Leader.h"

@implementation Team

- (id)init
{
    NSLog(@"Team init");
    self = [super init];
    if (self) {
        _leader = [[Leader alloc] init];
    }
    return self;
}

- (void)dealloc
{
    NSLog(@"Team dealloc");
    [_leader release];
    [super dealloc];
}


- (void)go
{
    NSLog(@"Team go");
    [_leader go];
}

- (void)changeLeader
{
    self.leader = [[[Leader alloc] init] autorelease];
}

@end
```

在类头文件中声明一个属性`@property (nonatomic, copy) NSString *name;`，编译器会自动为这个属性生成一个带有下划线的私有变量，
一个get方法和一个set方法。其中set方法比较复杂，它首先要释放已有变量，然后再根据指定的（retain, copy, assign）特性
来决定如何赋值。在Team中，`self.name = @"t1";`，基本上等同于如下的代码：
```
[_name release]; // 释放旧对象
_name = [@"t1" copy]; // copy方式
```

而`self.leader = [[[Leader alloc] init] autorelease];`，基本等同于：
```
[_leader release];
_leader = [[[Leader alloc] init] autorelease] retain]; // retain方式
```
注意上面的autorelease调用，因为leader属性是retain方式，如果不执行(auto)release，_leader指向的对象引用计数为2，
在最后析构函数中执行`[_leader release]`并不能释放这个对象。
另外在Team init函数中，在赋值时使用的代码是
```
_leader = [[Leader alloc] init];
```
这是因为在初始化函数执行时，_leader之前的值肯定为空，所以不需要清理，也就不需要调用属性set方法。
由于没有调用set方法，所以也就没有执行retain操作，所以也就不需要(auto)release了。

注意在init方法中使用autorelease，从而想省去在dealloc中release的麻烦的方法是错误的。比如
```
_leader = [[[Leader alloc] init] autorelease]
```
由于_leader被注册的内存池清理周期并不等同于Team对象的生存周期，所以，将造成内存错误。

对于Leader，有一个指向团队的属性是用assign方法的。这是因为如果这里也用retain方法，将造成循环引用，从而引起内存泄露。

1. 当一个对象拥有另外一个对象时，使用retain。如果对象支持NSCopying协议，也可以使用copy。
2. 当一个对象需要保存另外一个对象的指针，但并不拥有时，使用assign。
