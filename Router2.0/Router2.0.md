# Router2.0
## 在开始编码之前
我们在做router2.0的出发点是要做一个用于页面之间进行跳转通信的工具，解决目前二手车较为严重的耦合问题。

在router2.0之前，二手车项目中有着比较复杂的界面跳转工具交织在一起：lolita是二手车很久之前编码的一个通过url进行页面跳转的工具，支持拦截操作，功能单一；router1.0是为了桥街RN到原生之间的跳转，支持拦截操作，部分功能不完善，待改善，是一个比较典型的页面跳转通信工具；mediator是一个从target-action改造过来的一个纯中间层的跳转工具，将业务里的代码全部转移到mediator的各个扩展中，解决了耦合问题，功能较少。

二手车项目的现状算是一个三足鼎立的情况，router2.0的目的就是为了整合3个工具的优势功能并替换掉多余的工具。在做之前，我们还需要做一些额外的思考工作，router除了做单一的页面跳转通信外，我们还能赋予router这个工具其他具有实际意义的功能呢，并且不对router已经编码好的业务逻辑做过多处理？

* 做好从外部处理的跳转，如3d-touch，push-notification，深层多层跳转
* 同一家公司不同app之间的页面通信
* iOS、Android双端的协议统一
* RN、H5、Native 多端统一的可能性
* 没有接入JSPatch等热更新工具情况下的临时热更新修复方案
* 可配置，黑、白名单
* 在跳转通信之前的逻辑、安全性检查
* 一些统一的重复性的操作，多次跳转的埋点处理，runtime AOP处理
* router的介入能否改变现有的MVC设计模式，配合进行设计模式转变等等

一开始，我们只是想通过一种页面跳转，解决项目中的严重耦合性，随着调研的深入，我们发现类似的工具很多，并且实现方式也很不相同，大体总结为router路由类，protocol协议类和target-action的中间层类。不同类工具之间都有着自己的优势和劣势，最终本着做一款最适合自己项目的工具，选择了router类比较典型的工具进行编码工作。页面跳转工具的选择上，没有最完美的，只有最适合的。

## Router 2.0的源码解析

代码结构

```
UXIN_RouterManager
UXIN_Router
UXIN_RouterIntercepter
UXIN_RouterHandlerModel
UXIN_RouterTool
NSObject+KVCVerify
```
UXIN_RouterManager封装了对外所有可以调用的api，主要功能为打开URL，注册URL，拦截URL。

```
//打开处理URL
+ (void)openURL:(NSString *)URL;
...
//注册服务
+ (void)registerURLPattern:(NSString *)URLPattern toHandler:(UXIN_RouterHandler)handler;
...
//拦截URL
+ (void)interceptURLPattern:(NSString *)URLPattern priority:(NSInteger)priority toHandler:(UXIN_InterceptHandler)handler;
...

```
UXIN_Router的主要功能是处理URL和处理注册服务。注：注册服务的URL唯一，后者覆盖前者的注册服务操作。

```
/**
 *  打开此 URL
 *  会在已注册的 URL -> Handler 中寻找，如果找到，则执行 Handler
 *
 *  @param URL 带 Scheme，如 UXIN_://beauty/3
 */
+ (void)openURL:(NSString *)URL;
...
/**
 *  打开此 URL，带上附加信息，同时当操作完成时，执行额外的代码
 *
 *  @param URL        带 Scheme 的 URL，如 UXIN_://beauty/4
 *  @param userInfo 附加参数
 *  @param completion URL 处理完成后的 callback，完成的判定跟具体的业务相关
 */
+ (void)openURL:(NSString *)URL parameters:(NSDictionary *)parameters withUserInfo:(NSDictionary *)userInfo completion:(UXIN_RouterCompleteBlock)completion;
...
/**
 *  注册 URLPattern 对应的 Handler，在 handler 中可以初始化 VC，然后对 VC 做各种操作
 *
 *  @param URLPattern 带上 scheme，如 UXIN_://beauty/:id
 *  @param handler    该 block 会传一个字典，包含了注册的 URL 中对应的变量。
 *                    假如注册的 URL 为 UXIN_://beauty/:id 那么，就会传一个 @{@"id": 4} 这样的字典过来
 */
+ (void)registerURLPattern:(NSString *)URLPattern toHandler:(UXIN_RouterHandler)handler;
...
```

UXIN_RouterIntercepter处理拦截操作。注：拦截操作可以进行多级拦截，如拦截usedcar://home & usedcar://home/detail

```
/**
 添加拦截操作

 @param URLPattern 拦截URL
 @param priority 优先级，越低优先级越高 TODO：可能后续后调整
 @param handler 需要进行的操作
 */
+ (void)interceptURLPattern:(NSString *)URLPattern priority:(NSInteger)priority toHandler:(UXIN_InterceptHandler)handler;
...
/**
 获取所有拦截URL的操作

 @param URL url
 @param userInfo 用户信息
 @param params 参数
 @return 所有回调，数组格式
 */
+ (NSArray <UXIN_RouterInterceptCallBackModel *>*)getInterceptHandersWithURL:(NSString *)URL userInfo:(NSDictionary *)userInfo params:(NSDictionary *)params;
```

UXIN_RouterHandlerModel用作从处理URL开始到结束操作的封装对象进行通信使用。

```
//url完整路径
@property (nonatomic, copy) NSString *urlStr;

//从url中解析出来的参数,如果是可变路径,也会被解析到字典中
@property (nonatomic, strong) NSDictionary *params;

//openUrl方法中传入的userInfo
@property (nonatomic, strong) NSDictionary *userInfo;

//openUrl的完成回调,使用前请判断是否为nil
@property (nonatomic, copy) UXIN_RouterCompleteBlock completion;

//初始化后的实例对象
@property (weak, nonatomic) id instance;
```

UXIN_RouterToool封装了常用方法。

```
/**
 查找currentVC的上一个vc，如果为nil，为topVC

 @param currentVC 需要查找的vc
 @return 上一个vc
 */
+ (UIViewController *)prevViewController:(UIViewController *)currentVC;

+ (UIViewController *)topVC;

+ (UIViewController *)rootVC;

...
```

NSObject+KVCVerify用作参数传递之前的类型安全校验，参考了MJExtension实现方式。

```
/**
 对obj的属性赋值做判断,判断key是不是存在,value类型是否符合对象属性的类型,如果条件符合进行赋值,不符合抛出异常.

 @param value 赋值
 @param key 属性名称
 @param except 异常
 */
- (void)verifyPropertySetValue:(id)value forKey:(NSString *)key except:(void(^)(NSException *exp))except;

/**
 对obj的变量(包括属性)赋值做判断,判断key是不是存在,value类型是否符合对象属性的类型,如果条件符合进行赋值,不符合抛出异常.
 
 @param value 赋值
 @param key 属性名称
 @param except 异常
 */

- (void)verifySetValue:(id)value forKey:(NSString *)key except:(void(^)(NSException *exp))except;
```

其实本身router不具有页面跳转的功能，只是一个对URL进行多级拦截后进行相关的操作，注册与拦截在本质上没有区别，在代码逻辑上基本一样，只是不同操作保存在了不同的对象里，是为了业务在不同需求下的应用。

我们为二手车定制了一个符合我们业务需求的路由协议解析规则，如：scheme://classname.action?parameters_query，在业务中默认实现了一个拦截操作，针对实现上述协议的url来说，不需要做注册服务和拦截服务，在默认操作中将class类名、action行为、parameters参数进行解析后，尝试进行初始化，参数传递，页面跳转等行为。

如果用户实现了拦截操作，如对usedcar://ViewController.push?id=123进行了拦截操作，在Intercepter中的数据存储结构改变如下

```
{
    usedcar =     {
        "ViewController.push" =         {
            "_" =             (
                "<UXIN_RouterInterceptPriorityModel: 0x2835e7ee0>"
            );
        };
        "~" =         {
            "_" =             (
                "<UXIN_RouterInterceptPriorityModel: 0x2835c2fe0>"
            );
        };
    };
}
```

如果开发者在业务端实现了注册服务，也会在Router的对象中有类似的存储结构。我们的优先级顺序是这样的：

* 没有拦截，没有注册服务，执行默认操作
* 有拦截（可多级多次拦截），没有注册服务，先执行业务端的拦截操作，再执行默认操作
* 没有拦截，有注册服务，执行注册服务，不再执行默认操作
* 有拦截（可多级多次拦截），有注册服务，先执行业务端的拦截操作，再执行注册服务，不再执行默认操作

以上的处理逻辑，则满足了二手车目前所有的页面跳转通信需求


## 迁移、替换过程

在Mediator进行整合的所有页面中，有2种情况分别进行处理：

YXUCarDetailVC2 车辆详情页、YXU_CarMarketListVC 车市列表页 push进入情况、YXUCityListVC 城市列表选择、YXUThousandCountryListVC 千县列表选择、EvaluateViewController 卖车估价页面、UXUMessageCneterVC 消息中心、YXUHomeSearchVC 搜索页面，有回调 都可以通过协议拼接，执行默认操作进行替换

YXU_CarMarketListVC tabbar切换 车市列表、YXUEMManager 客服页面，特殊初始化和push、YXU_LoginBridge 登录页面，非controller 通过注册服务，实现特殊的页面通信，从而不执行默认操作进行了相关替换。

RN部分的代码更改了api的调用，因为RN端的url协议规则和Router2.0的规则没有统一，全部采用的注册服务形式。

Lolita的代码涉及面非常广，仍然在替换种，目前没有遇到问题。

## 总结
之前无论对router还是相关工具理解的太狭隘了，发现类似的工具在较大项目中是不可或缺的。在对router的编码过程中，一直对router在安全、高效、多功能、便捷等方面寻求一个平衡点。总之router2.0只是确立了一个方向，编码工作还会继续进行。

## 参考
[iOS 组件化 —— 路由设计思路分析 - LPD-iOS](https://lpd-ios.github.io/2017/02/26/iOS-Router/)

[蘑菇街 App 的组件化之路](https://limboy.me/tech/2016/03/10/mgj-components.html)

[滴滴的组件化实践与优化](https://www.infoq.cn/article/xiaojukeji-component-practice-and-optimization?useSponsorshipSuggestions=true)