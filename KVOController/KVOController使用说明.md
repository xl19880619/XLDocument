# KVOController使用说明

简单举例

```
 [self.KVOController 
 observe:target 
 keyPath:@"keypath" 
 options: NSKeyValueObservingOptionInitial | 
 		  NSKeyValueObservingOptionNew ... 
 block:^(id  _Nullable observer, id  _Nonnull object, NSDictionary<NSString *,id> * _Nonnull change) {
        NSLog(@"change");
        //do something
    }];
```

KVO 作为 iOS 中一种强大并且有效的机制，为 iOS 开发者们提供了很多的便利；我们可以使用 KVO 来检测对象属性的变化、快速做出响应，这能够为我们在开发强交互、响应式应用以及实现视图和模型的双向绑定时提供大量的帮助。

#### 为什么要使用KVOController

- 使用带Block的KVO调用，提升使用 KVO 的体验
- 代码块清晰，实现 KVO 与事件发生处的代码上下文相同，不需要跨方法传参数
- 每一个 keyPath 会对应一个属性，不需要在 block 中使用 if 判断 keyPath
- 方便解决多级页面响应回传，避免多级block
- 封装了一层系统api，调用方便，不需要关心removeObserver
- balabalabala说了那么多，反正我们用了

#### 使用过程中注意点问题

问题还是有点的，但是比原生调用强点的

- 调用KVO的Block里，必须使用weakSelf/strongSelf，不可以使用super，_var写法，否则造成循环引用 eg：

```
@weakify(self);
[self.KVOController observe:self.viewModel.loadDataCommand
                    keyPath:@"result"
                    options:NSKeyValueObservingOptionNew
                      block:^(id observer, id object, NSDictionary *change) {
    @strongify(self);
    CommandResult *result = change[NSKeyValueChangeNewKey];
    if (result.error) {
        [super loadDataFailure:result.error];
    } else{
        _headerView.segmentIndex = self.jumptonew ? 1 : 0;
        _navBarVC.barImageEntity = self.viewModel.shopHeadEntity.topbarImg;
    }
}];
```

- 当target是self本身时，需要使用 ```self.KVOControllerNonRetaining observe:self``` 或者直接在self的setter里进行数据更新，否则造成循环引用 eg：

```
@weakify(self);
[self.KVOController observe:self
                    keyPath:@"emojiViewStatus"
                    options:0
                      block:^(id observer, id object, NSDictionary *change) {
    @strongify(self);
    [self dealEmojiViewWithEmojiStatus:object];
}];
```

- 只能使用 self.kvo 持有进行调用，demo中的临时变量不能捕捉kvo变化，FBKVOController 会直接调用析构函数

```
// create KVO controller with observer
FBKVOController *KVOController = [FBKVOController controllerWithObserver:self];

// observe clock date property
[KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew block:^(ClockView *clockView, Clock *clock, NSDictionary *change) {

  // update clock view with new value
  clockView.date = change[NSKeyValueChangeNewKey];
}];
```

- change对象需要进行多重安全性判断，是否有值，并且是否属于某对象类型

```
if (change[NSKeyValueChangeNewKey]) {
	if ([change[NSKeyValueChangeNewKey] isKindOfClass:name.class]) {
		//do logic code
	} else {
		//null or other
	}
}
```