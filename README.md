# WJRuntimeSummary
#Runtime简单粗暴理解

 从C的面向过程到接触OC的对象、消息的过渡初期总会有知其然不知其所以然的纠结，相关的学习资源一般都是介绍有什么、使用步骤一二三四的套路，这样就很难知道知道本质是什么，能干什么不能干什么，为什么要选择用它。而实际开发过程，都是先有什么要解决，再努力找到实现方法。人脑的容易接受的信息，也多是主干到分枝的思维导图，纲举目张。所以，试着以自己的粗浅理解来写一点关于OC运行时的东西。

代码的思想，大概是把重复且不变的东西封装成可以重复利用的共性，把变化的东西细化为具体独立松耦合的变量。这些可以是数据类型，也可以是实现的方法代码片段。类也是封装的产物和可封装的对象。被封装的东西，需要找到里面内容来具体地实现，就需要给里面内容加个关联的映射标识，比如索引（数组）、字符串（字典）、指针、SEL（方法的代号）、isa（对象）等等。大概来说就是用类和对象来封装父类指针和方法列表，用映射来找到实现方法的代码片段。

##一、运行时Runtime介绍
作用：在程序运行的时候执行编译后的代码，可以:

      > 动态（创建）、(修改)、(内省)  `类`和`方法`
      
      > 消息传递
      
运行时Runtime的一切都围绕这两个中心：类的动态配置 和 消息传递。通过操作函数来配置类信息，通过msgSend函数传递消息。
本质：libobjc.dylib，C和汇编（消息传递机制由汇编写成）写成。

##二、类的本质：
###1、类相关：
数据结构（本源）:Class类型的结构体。在objc/runtime.h中查看其成员：

struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

/#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类

    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识

    long instance_size                      OBJC2_UNAVAILABLE;  // 类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 类的成员变量链表

    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存

    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
/#endif

} OBJC2_UNAVAILABLE;

####a、数据类型：
isa和super_class ：不同的类中可以有相同的方法（同一个类的方法不能同名，哪怕参数类型不同，后面解释...），所以要先确定是那个类。isa和super_class是找到实现函数的关键映射，决定找到存放在哪个类的方法实现。（isa用于自省确定所属类，super_class确定继承关系）。

实例对象的isa指针指向类，类的isa指针指向其元类（metaClass）。对象就是一个含isa指针的结构体。类存储实例对象的方法列表，元类存储类的方法列表，元类也是类对象。
这是id类型的结构（类似于C里面的void *）：

struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

typedef struct objc_object *id;

当创建实例对象时，分配的内存包含一个objc_object数据结构，然后是类到父类直到根类NSObject的实例变量的数据。NSObject类的alloc和allocWithZone:方法使用函数class_createInstance来创建objc_object数据结构。

向一个Objective-C对象发送消息时，运行时库会根据实例对象的isa指针找到这个实例对象所属的类。Runtime库会在类的方法列表由super_class指针找到父类的方法列表直至根类NSObject中去寻找与消息对应的selector指向的方法。找到后即运行这个方法。

![image](https://github.com/WinJayQ/WJRuntimeSummary/raw/master/WJRuntimeSummary/1.jpg) 

                         metaClass.png
                         
上图是关于isa和super_class指针的图解：
1、isa：实例对象->类->元类->（不经过父元类）直接到根元类（NSObject的元类），根元类的isa指向自己；

2、superclass：类->父类->...->根类NSObject，元类->父元类->...->根元类->根类,NSObject的superclass指向nil。

####b、操作函数：类对象以class_为前缀，实例对象以object_为前缀
####class_：

#####get: 类名，父类，元类；实例变量，成员变量；属性；实例方法，类方法，方法实现；

// 获取类的类名
const char * class_getName ( Class cls );

// 获取类的父类
Class class_getSuperclass ( Class cls );

// 获取实例大小
size_t class_getInstanceSize ( Class cls );

// 获取类中指定名称实例成员变量的信息
Ivar class_getInstanceVariable ( Class cls, const char *name );

// 获取类成员变量的信息
Ivar class_getClassVariable ( Class cls, const char *name );

// 获取指定的属性
objc_property_t class_getProperty ( Class cls, const char *name );

// 获取实例方法
Method class_getInstanceMethod ( Class cls, SEL name );

// 获取类方法
Method class_getClassMethod ( Class cls, SEL name );

// 获取方法的具体实现
IMP class_getMethodImplementation ( Class cls, SEL name );
IMP class_getMethodImplementation_stret ( Class cls, SEL name );

#####copy: 成员变量列表；属性列表；方法列表；协议列表；

// 获取整个成员变量列表
Ivar * class_copyIvarList ( Class cls, unsigned int *outCount );

// 获取属性列表
objc_property_t * class_copyPropertyList ( Class cls, unsigned int *outCount );

// 获取所有方法的列表
Method * class_copyMethodList ( Class cls, unsigned int *outCount );

// 获取类实现的协议列表
Protocol * class_copyProtocolList ( Class cls, unsigned int *outCount );

#####add: 成员变量；属性；方法；协议；(添加成员变量只能在运行时创建的类，且不能为元类)

// 添加成员变量

BOOL class_addIvar ( Class cls, const char *name, size_t size, uint8_t alignment, const char *types );

// 添加属性
BOOL class_addProperty ( Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount );

// 添加方法
BOOL class_addMethod ( Class cls, SEL name, IMP imp, const char *types );

// 添加协议
BOOL class_addProtocol ( Class cls, Protocol *protocol );

#####replace：属性；方法；

// 替换类的属性
void class_replaceProperty ( Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount );

// 替代方法的实现
IMP class_replaceMethod ( Class cls, SEL name, IMP imp, const char *types );

#####respond:响应方法判断（内省）

// 类实例是否响应指定的selector
BOOL class_respondsToSelector ( Class cls, SEL sel );

#####isMetaClass:元类判断（内省）

// 判断给定的Class是否是一个元类
BOOL class_isMetaClass ( Class cls );

#####conform：遵循协议判断（内省）

// 返回类是否实现指定的协议
BOOL class_conformsToProtocol ( Class cls, Protocol *protocol );

####objc_：
#####get: 实例变量；成员变量；类名；类；元类；关联对象

// 获取对象实例变量
Ivar object_getInstanceVariable ( id obj, const char *name, void **outValue );

// 获取对象中实例变量的值
id object_getIvar ( id obj, Ivar ivar );

// 获取对象的类名
const char * object_getClassName ( id obj );

// 获取对象的类
Class object_getClass ( id obj );
Class objc_getClass ( const char *name );

// 返回指定类的元类
Class objc_getMetaClass ( const char *name );

//获取关联对象
id objc_getAssociatedObject(self, &myKey);

#####copy:对象；类；类列表；协议列表；

// 获取指定对象的一份拷贝
id object_copy ( id obj, size_t size );
// 创建并返回一个指向所有已注册类的指针列表
Class * objc_copyClassList ( unsigned int *outCount );

#####set: 实例变量；类；类列表；协议；关联对象；

// 设置类实例的实例变量的值
Ivar object_setInstanceVariable ( id obj, const char *name, void *value );

// 设置对象中实例变量的值
void object_setIvar ( id obj, Ivar ivar, id value );

//设置关联对象
void objc_setAssociatedObject(self, &myKey, anObject, OBJC_ASSOCIATION_RETAIN);

#####dispose: 对象；

// 释放指定对象占用的内存
id object_dispose ( id obj );

动态创建/销毁类、对象
动态创建/销毁类：
// 创建一个新类和元类
Class objc_allocateClassPair ( Class superclass, const char *name, size_t extraBytes );

// 销毁一个类及其相关联的类
void objc_disposeClassPair ( Class cls );

// 在应用中注册由objc_allocateClassPair创建的类
void objc_registerClassPair ( Class cls );

动态创建/销毁对象：
// 创建类实例
id class_createInstance ( Class cls, size_t extraBytes );

// 在指定位置创建类实例
id objc_constructInstance ( Class cls, void *bytes );

// 销毁类实例
void * objc_destructInstance ( id obj );

###2、实例变量、属性相关：
实例变量和属性也是类对象的关键配置。
属性变量的意义就是方便让其他对象访问实例变量，另外可以拓展实例变量的作用范围。当然，你可以设置只读或者可写等，设置方法也可自定义。
####a、数据类型：
Ivar；
typedef struct objc_ivar *Ivar;

struct objc_ivar {
    char *ivar_name                 OBJC2_UNAVAILABLE;  // 变量名
    char *ivar_type                 OBJC2_UNAVAILABLE;  // 变量类型
    int ivar_offset                 OBJC2_UNAVAILABLE;  // 基地址偏移字节
/#ifdef __LP64__
    int space                       OBJC2_UNAVAILABLE;
/#endif
}
objc_property_t(取名可能是因为当时Objective-C1.0还没属性)；
typedef struct objc_property *objc_property_t;
objc_property_attribute_t（属性的特性有：返回值、是否为atomic、getter/setter名字、是否为dynamic、背后使用的ivar名字、是否为弱引用等）；
typedef struct {
    const char *name;           // 特性名
    const char *value;          // 特性值
} objc_property_attribute_t;

####b、操作函数：
ivar_：
#####get：
// 获取成员变量名
const char * ivar_getName ( Ivar v );

// 获取成员变量类型编码
const char * ivar_getTypeEncoding ( Ivar v );

// 获取成员变量的偏移量
ptrdiff_t ivar_getOffset ( Ivar v );

#####property_：
// 获取属性名
const char * property_getName ( objc_property_t property );

// 获取属性特性描述字符串
const char * property_getAttributes ( objc_property_t property );

// 获取属性中指定的特性
char * property_copyAttributeValue ( objc_property_t property, const char *attributeName );

// 获取属性的特性列表
objc_property_attribute_t * property_copyAttributeList ( objc_property_t property, unsigned int *outCount );

###3、 方法消息相关：
消息传递机制是Runtime的核心，也即消息分派器objc_msgSend。先要知道几个概念。
####a、 数据类型：
#####SEL；
SEL又叫选择器，是表示一个方法的selector的指针,映射方法的名字。Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)，这个标识就是SEL。
SEL的作用是作为IMP的KEY，存储在NSSet中，便于hash快速查询方法。SEL不能相同，对应方法可以不同。所以在Objective-C同一个类(及类的继承体系)中，不能存在2个同名的方法，就算参数类型不同。多个方法可以有同一个SEL。
不同的类可以有相同的方法名。不同类的实例对象执行相同的selector时，会在各自的方法列表中去根据selector去寻找自己对应的IMP。
相关概念：类型编码（Type Encoding）
编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的selector关联在一起。可以使用@encode编译器指令来获取它。
typedef struct objc_selector *SEL;
<objc/runtime.h>中没有公开具体的objc_selector结构体成员。但通过log可知SEL本质是一个字符串。

#####IMP;
IMP是指向实现函数的指针，通过SEL取得IMP后，我们就获得了最终要找的实现函数的入口。

typedefine id (*IMP)(id, SEL, ...)

#####Method；
这个结构体相当于在SEL和IMP之间作了一个绑定。这样有了SEL，我们便可以找到对应的IMP，从而调用方法的实现代码。（在运行时才将SEL和IMP绑定, 动态配置方法）

typedef struct objc_method *Method;

struct objc_method {
    SEL method_name                 OBJC2_UNAVAILABLE;  // 方法名
    char *method_types                  OBJC2_UNAVAILABLE; // 参数类型
    IMP method_imp                      OBJC2_UNAVAILABLE;  // 方法实现
}

objc_method_list 就是用来存储当前类的方法链表，objc_method存储了类的某个方法的信息。

struct objc_method_list {
    struct objc_method_list *obsolete                        OBJC2_UNAVAILABLE;
    int method_count                                                 OBJC2_UNAVAILABLE;
/#ifdef __LP64__
    int space                                                              OBJC2_UNAVAILABLE;
/#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}


#####方法缓存；
方法调用最先是在方法缓存里找的，方法调用是懒调用，第一次调用时加载后加到缓存池里。一个objc程序启动后，需要进行类的初始化、调用方法时的cache初始化，再发送消息的时候就直接走缓存（引申：+load方法和+initialize方法。load方法是首次加载类时调用，绝对只调用一次；initialize方法是首次给类发消息时调用，通常只调用一次，但如果它的子类初始化时未定义initialize方法，则会再调用一次它的initialize方法）。

struct objc_cache {
    // 缓存bucket的总数
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;

    // 实际缓存bucket的总数
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    // 指向Method数据结构指针的数组
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};


####b、 操作函数:
#####method_：
invoke: 方法实现的返回值；

// 调用指定方法的实现
id method_invoke ( id receiver, Method m, ... );

// 调用返回一个数据结构的方法的实现
void method_invoke_stret ( id receiver, Method m, ... );

#####get: 方法名；方法实现；参数与返回值相关；


// 获取方法名
SEL method_getName ( Method m );

// 返回方法的实现
IMP method_getImplementation ( Method m );
// 获取描述方法参数和返回值类型的字符串
const char * method_getTypeEncoding ( Method m );
// 返回方法的参数的个数
unsigned int method_getNumberOfArguments ( Method m );
// 通过引用返回方法指定位置参数的类型字符串
void method_getArgumentType ( Method m, unsigned int index, char *dst, size_t dst_len );


#####copy: 返回值类型，参数类型

// 获取方法的返回值类型的字符串
char * method_copyReturnType ( Method m );

// 获取方法的指定位置参数的类型字符串
char * method_copyArgumentType ( Method m, unsigned int index );

// 通过引用返回方法的返回值类型字符串
void method_getReturnType ( Method m, char *dst, size_t dst_len );


#####set：方法实现；

// 设置方法的实现
IMP method_setImplementation ( Method m, IMP imp );


#####exchange：交换方法实现

// 交换两个方法的实现
void method_exchangeImplementations ( Method m1, Method m2 );

#####description : 方法描述

// 返回指定方法的方法描述结构体
struct objc_method_description * method_getDescription ( Method m );


####sel_

// 返回给定选择器指定的方法的名称
const char * sel_getName ( SEL sel );

// 在Objective-C Runtime系统中注册一个方法，将方法名映射到一个选择器，并返回这个选择器
SEL sel_registerName ( const char *str );

// 在Objective-C Runtime系统中注册一个方法
SEL sel_getUid ( const char *str );

// 比较两个选择器
BOOL sel_isEqual ( SEL lhs, SEL rhs );


####c、方法调用流程：
向对象发送消息，实际上是调用objc_msgSend函数，obj_msgSend的实际动作就是：找到这个函数指针，然后调用它。

id objc_msgSend(receiver self, selector _cmd, arg1, arg2, ...)

self和_cmd是隐藏参数，在编译期被插入实现代码。
self：指向消息的接受者target的对象类型，作为一个占位参数，消息传递成功后self将指向消息的receiver。
_cmd: 指向方法实现的SEL类型。

当向一般对象发送消息时，调用objc_msgSend；当向super发送消息时，调用的是objc_msgSendSuper； 如果返回值是一个结构体，则会调用objc_msgSend_stret或objc_msgSendSuper_stret。


0.1-检查target是否为nil。如果为nil，直接cleanup，然后return。(这就是我们可以向nil发送消息的原因。)

如果方法返回值是一个对象，那么发送给nil的消息将返回nil；如果方法返回值为指针类型，其指针大小为小于或者等于sizeof(void*)，float，double，long double 或者long long的整型标量，发送给nil的消息将返回0；如果方法返回值为结构体,发送给nil的消息将返回0。结构体中各个字段的值将都是0；如果方法的返回值不是上述提到的几种情况，那么发送给nil的消息的返回值将是未定义的。

0.2-如果target非nil，在target的Class中根据Selector去找IMP。（因为同一个方法可能在不同的类中有不同的实现，所以我们需要依赖于接收者的类来找到的确切的实现）。

1-首先它找到selector对应的方法实现:

*1.1-在target类的方法缓存列表里检查有没有对应的方法实现，有的话，直接调用。

*1.2-比较请求的selector和类方法列表中的selector，对应的话，直接调用。

*1.3-比较请求的selector和父类方法列表，父类的父类，直至根类，如果有对应，则直接调用。（方法重写拦截父类方法的原理）

2-调用方法实现，并将接收者对象及方法的所有参数传给它。

3-最后，将实现函数的返回值作为自己的返回值。


####d、动态方法解析与消息转发：
如果以上的类中没有找到对应的selector（一般保险起见先用respondsToSelector:内省判断）：，还可以利用消息转发机制依次执行以下流程：

#####Method Resolution（动态方法解析）：
用所属类的类方法+（BOOL）resolveInstanceMethod:(实例方法)或者+（BOOL）resolveClassMethod:(类方法),在此方法里添加class_addMethod函数。一般用于@dynamic动态属性。（当一个属性声明为@dynamic，就是向编译器保证编译时不用管/get实现，一定会在运行时实现）。
Fast Forwarding （快速消息转发）：
如果上一步无法响应消息，调用- (id)forwardingTargetForSelector:(SEL)aSelector方法，将消息接受者转发到另一个对象target（不能为self，否则死循环）。

#####Normal Forwarding（普通消息转发）：
如果上一步无法响应消息：
调用方法签名- (NSMethodSignature )methodSignatureForSelector:(SEL)aSelector，方法签名目的将函数的参数类型和返回值封装；
如果返回非nil，则创建一个NSInvocation对象利用方法签名和selector封装未被处理的消息，作为参数传递给- (void)forwardInvocation:(NSInvocation )anInvocation。

这一步比较耗时。
如果以上步骤（消息传递和消息转发）还是不能响应消息，则调动doesNotRecognizeSelector：方法，抛出异常。
unrecognized selector sent to instance
(消息转发可以利用转移消息接受对象，实现伪多重继承的效果。)

###4、 协议相关：@protocol声明了可以被其他任何类实现的方法，协议仅仅是定义一个接口，而由其他的类去负责实现。
####数据类型：Protocol；

typedef struct objc_object Protocol;

protocol是一个对象结构体。

####操作函数：

#####objc_:
// 返回指定的协议
Protocol * objc_getProtocol ( const char *name );

// 获取运行时所知道的所有协议的数组
Protocol ** objc_copyProtocolList ( unsigned int *outCount );

// 创建新的协议实例
Protocol * objc_allocateProtocol ( const char *name );

// 在运行时中注册新创建的协议
void objc_registerProtocol ( Protocol *proto );

#####protocol_：
#####get: 协议；属性；

// 返回协议名
const char * protocol_getName ( Protocol *p );
// 获取协议的指定属性
objc_property_t protocol_getProperty ( Protocol *proto, const char *name, BOOL isRequiredProperty, BOOL isInstanceProperty );

#####copy：协议列表；属性列表；

// 获取协议中的属性列表
objc_property_t * protocol_copyPropertyList ( Protocol *proto, unsigned int *outCount );
// 获取协议采用的协议
Protocol ** protocol_copyProtocolList ( Protocol *proto, unsigned int *outCount );

#####add：属性；方法；协议；

// 为协议添加方法
void protocol_addMethodDescription ( Protocol *proto, SEL name, const char *types, BOOL isRequiredMethod, BOOL isInstanceMethod );

// 添加一个已注册的协议到协议中
void protocol_addProtocol ( Protocol *proto, Protocol *addition );

// 为协议添加属性
void protocol_addProperty ( Protocol *proto, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount, BOOL isRequiredProperty, BOOL isInstanceProperty );


#####isEqual：判断两协议等同；

// 测试两个协议是否相等
BOOL protocol_isEqual ( Protocol *proto, Protocol *other );

#####comform：判断是否遵循协议；

// 查看协议是否采用了另一个协议
BOOL protocol_conformsToProtocol ( Protocol *proto, Protocol *other );

###5、 其他：类名；版本号；类信息；（忽略）


##三、 动态实现：
###Method Swizzling;
Method Swizzling可以在运行时通过修改类的方法列表中selector对应的函数或者设置交换方法实现，来动态修改方法。可以重写某个方法而不用继承，同时还可以调用原先的实现。通常应用于在category中添加一个方法。
为保证改变方法引起冲突，确保方法混用只能一次性：
比如，在+load方法或者dispatch_once中执行。
ISA Swizzling；
ISA Swizzling可以动态修改对象的isa指针，改变对象的类，类似于创建子类实现相同的功能。KVO即是同过ISA Swizzling实现的。


##四、 其他概念：category；super；
###category:

typedef struct objc_category *Category;

struct objc_category {
    char *category_name                          OBJC2_UNAVAILABLE; // 分类名
    char *class_name                             OBJC2_UNAVAILABLE; // 分类所属的类名
    struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE; // 实例方法列表
    struct objc_method_list *class_methods       OBJC2_UNAVAILABLE; // 类方法列表
    struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}  

// objc-runtime-new.h中定义：
struct category_t {
    const char *name;                        // name 是指 class_name 而不是 category_name
    classref_t cls;                          // cls是要扩展的类对象，编译期间是不会定义的，而是在Runtime阶段通过name对应到对应的类对象
    struct method_list_t *instanceMethods;       
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;    // instanceProperties表示Category里所有的properties，(这就是我们可以通过objc_setAssociatedObject和objc_getAssociatedObject增加实例变量的原因，)不过这个和一般的实例变量是不一样的

};


category就是定义方法的结构体，instance_methods列表是objc_class中方法列表的一个子集，class_methods列表是元类方法列表的一个子集。由其结构成员可知，category为什么不能添加成员变量（可添加属性，只有set/get方法）。

给category添加方法后，category_list会生成method list。这个方法列表是倒序添加的，也就是说，新生成的category的方法会先于旧的category的方法插入。（category的方法会优先于类方法执行）。


###super：
super并不是隐藏参数，它实际上只是一个”编译器标示符”，它负责告诉编译器，当调用方法时，跳过当前类去调用父类的方法，而不是本类中的方法。self是类的一个隐藏参数，每个方法的实现的第一个参数即为self。实际上给super发消息时，super还是与self指向的是相同的消息接收者。

struct objc_super {
   __unsafe_unretained id receiver;
   __unsafe_unretained Class super_class;
};

原理：使用super来接收消息时，编译器会生成一个objc_super结构体。发送消息时，不是调用objc_msgSend函数，而是调用objc_msgSendSuper函数:

id objc_msgSendSuper ( struct objc_super *super, SEL op, ... );

该函数实际的操作是：从objc_super结构体指向的superClass的方法列表开始查找selector，找到后以objc->receiver去调用这个selector。


###Runtime开源源码对一些方法的实现：

- (Class)class ;

- (Class)class {
    return object_getClass(self);
}

+ (Class)class;

+ (Class)class {
    return self;
}

- (BOOL)isKindOf:aClass;// (for循环遍历父类，每次判断返回的结果可能不同)

- (BOOL)isKindOf:aClass
{
    Class cls;
    for (cls = isa; cls; cls = cls->superclass) 
        if (cls == (Class)aClass)
            return YES;
    return NO;
}

- (BOOL)isMemberOf:aClass;

- (BOOL)isMemberOf:aClass
{
    return isa == (Class)aClass;
}


#####转自楚天舒的简书-->http://www.jianshu.com/p/f900de4a1495
