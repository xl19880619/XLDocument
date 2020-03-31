# iOS性能优化分享

好的app应该具有优秀的体验，在我们的日常开发中，可能会更注重功能的开发，随着技术的发展，iPhone手机的性能越来越好，我们会比较少的考虑性能的问题。在对app内列表页及详情页分析，发现在个别手机个别系统下会有卡顿情况的出现。以下是我针对性能优化卡顿方面做的总结，在此分享一下。

## 一、卡顿的原因

### 1.1 成像原理：

在iOS中采用双缓冲机制+垂直信号+水平信号达到屏幕成像显示效果。

双缓冲机制分为前帧缓存、后帧缓存，GPU 会预先渲染好一帧放入一个缓冲区内（前帧缓存），让视频控制器读取，当下一帧渲染好后，GPU 会直接把视频控制器的指针指向第二个缓冲器（后帧缓存），当视频控制器按照水平信号指示读完一帧准备读下一帧时，GPU会等待显示器的垂直信号，垂直信号发出后，前帧缓存与后帧缓存进行切换。

### 1.2卡顿原因

简单来说，在垂直信号到来后，CPU会计算需要显示的内容，如创建视图控件、布局计算、文本绘制等，然后GPU会把CPU计算好的内容经过合成渲染等操作，提交到缓冲区内，等到下一次垂直信号到来时显示。如果在下一次垂直信号到来时，CPU和GPU没有完成内容的提交，则这一帧就会被丢弃，等到下一次垂直信号的到来再显示，也就造成了界面卡顿。

通过以上原理可以知道图像显示是由CPU和GPU共同完成的，那么优化一般也遵循这两个原则：

（1）减少CPU和GPU的负载

（2）均衡使用CPU和GPU

### 1.3 CPU消耗型操作

#### 1.3.1布局计算

布局计算应该是iOS中最常见的CPU消耗资源的地方了，如果视图层级复杂，计算出所有图层的布局信息就会消耗一部分时间。还要避免不必要的更新布局。

#### 1.3.2对象的创建

应避免多余的对象的创建，属性的修改，对象创建会伴随着内存分配、属性设置，甚至还有读取文件等操作，会比较消耗CPU资源。

延迟对象创建的时间，一开始并不需要显示的对象可以采用懒加载延迟创建等。

#### 1.3.3 Autolayout布局

对于复杂视图，使用过多的约束布局会造成严重的性能问题。如masonry：make<update<remake。

#### 1.3.4文本计算

如果界面中包含大量文本，文本的计算会占用较大CPU资源等。

#### 1.3.5 文本渲染

屏幕上能看到的所有文本内容控件，包括 UIWebView，在底层都是通过 CoreText 排版、绘制为 Bitmap 显示的。常见的文本控件 （UILabel、UITextView 等），其排版和绘制都是在主线程进行的，当显示大量文本时，CPU 的压力会非常大。

#### 1.3.6图片解码及图像绘制

当使用 UIImage 或 CGImageSource 的那几个方法创建图片时，图片数据并不会立刻解码。图片设置到 UIImageView 或者 CALayer.contents 中去，并且 CALayer 被提交到 GPU 前，CGImage 中的数据才会得到解码。这一步是发生在主线程的，并且不可避免。如果想要绕开这个机制，常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片。目前常见的网络图片库都自带这个功能。

### 1.4 GPU消耗型操作

#### 1.4.1纹理的渲染

所有的 Bitmap，包括图片、文本、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。当在较短时间显示大量图片时（比如 TableView 存在非常多的图片并且快速滑动时），CPU 占用率很低，GPU 占用非常高，界面仍然会掉帧。

当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 和 GPU 都会带来额外的资源消耗。

#### 1.4.2视图的混合 (Blended)

当多个视图（或者说 CALayer）重叠在一起显示时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，混合的过程也会消耗很多 GPU 资源。为了减轻这种情况的 GPU 消耗，应用应当尽量减少视图数量和层次，并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。当然，这也可以用上面的方法，把多个视图预先渲染为一张图片来显示。

#### 1.4.3图形的生成

CALayer 的 border、圆角、阴影、遮罩（mask），CASharpLayer 的矢量图形显示，通常会触发离屏渲染（offscreen rendering），而离屏渲染通常发生在 GPU 中。当一个列表视图中出现大量圆角的 CALayer，并且快速滑动时，可以观察到 GPU 资源已经占满，而 CPU 资源消耗很少。这时界面仍然能正常滑动，但平均帧数会降到很低。为了避免这种情况，可以尝试开启 CALayer.shouldRasterize 属性，但这会把原本离屏渲染的操作转嫁到 CPU 上去。避免使用圆角、阴影、遮罩等属性。

##### 扩展：离屏渲染消耗性能的原因

在VSync(垂直脉冲)信号作用下，视频控制器每隔16.67ms就会去帧缓冲区(当前屏幕缓冲区)读取渲染后的数据；但是有些效果被认为不能直接呈现于屏幕前，而需要在别的地方做额外的处理，进行预合成。比如图层属性的混合体再没有预合成之前不能直接在屏幕中绘制，所以就需要屏幕外渲染。

###### 消耗性能的主要原因包括：

a.需要创建新的缓冲区

b.离屏渲染的整个过程：需要多次切换上下文环境，先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上，又需要将上下文环境从离屏切换到当前屏幕。

###### 哪些操作会触发离屏渲染？

a. 光栅化，layer.shouldRasterize = YES，当一个图像混合了多个图层，每次移动时，每一帧都要重新合成这些图层，十分消耗性能。当我们开启光栅化后，会强制开启离屏渲染的同时，并在首次生成一个位图缓存，当再次使用时候就会复用这个缓存，可以避免多次渲染的开销。

但是不要过度使用光栅化，缓存区的大小被设置为屏幕大小的2.5倍，假如过分使用同样会导致大量的离屏渲染。如果缓存的内容超过100ms没有被使用则会被回收。

b.遮罩，layer.mask

c.圆角，同时设置layer.masksToBounds = YES、layer.cornerRadius大于0

考虑通过CoreGraphics绘制裁剪圆角，或者让UI提供圆角图片

d.阴影，layer.shadow

### 1.5 当遇到性能问题时，应该从哪些方面思考呢？

是否有不必要的CPU渲染？

是否有太多的离屏渲染操作？

是否有太多的图层混合操作？

是否有奇怪的图片格式或者尺寸？

是否涉及到复杂的view或者效果？

## 二、工具

### 2.1 instruments

#### 2.1.1 Core Animation

操作方式：手指不离开屏幕，快速上下滑动屏幕。值60最佳，按照苹果官方说法，当低于45时，用户会感觉到明显的卡顿。

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/coreAnimation.png)


#### 2.1.2 Time Profiler

主要检查方法耗时情况

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/timeProfiler.png)

Separate by State：此选项会根据应用程序的生命周期状态对结果进行分组。

Separate by Thread：通过线程分类来查看那些方法占用CPU最多。

Invert Call Tree：调用树倒返过来，将习惯性的从根向下一级一级的显示，如选上就会返过来从最底层调用向一级一级的显示。如果想要查看那个方法调用为最深时使用会更方便些。

Hide Missing Symbols：隐藏丢失的符号，比如应用或者系统的dSYM文件找不到的话，在详情面板上是看不到方法名的，只能看一些读不明的十六进值，所以对我们来说是没有意义的，去掉了会使阅读更清楚些。

Hide System Libraries：选上它只会展示与应用有关的符号信息，一般情况下我们只关心自己写的代码所需的耗时，而不关心系统库的CPU耗时。

Flatten Recursion：选上它会将调用栈里递归函数作为一个入口。

Top Functions：选上它会将最耗时的函数降序排列，而这种耗时是累加的，比如A调用了B，那么A的耗时数是会包含B的耗时数。

### 2.2 基于CADisplayLink来测量帧率。

原理：CADisplayLink是一个能让我们以和屏幕刷新率相同的频率将内容画到屏幕上的定时器，在以特定的mode加入runloop时，每次屏幕内容刷新结束时，runloop就会向对应的target发送一次selector方法，selector就会被调用一次。既然CADisplayLink可以以屏幕刷新的频率调用指定selector，而且iOS系统中正常的屏幕刷新率为60Hz（60次每秒），那只要在这个方法里面统计每秒这个方法执行的次数，通过次数/时间就可以得出当前屏幕的刷新率了。

### 2.3 View debugging ->Rendering

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/viewRendering.png)

#### 2.3.1 Color Blended Layers（混合过度绘制）

这个选项基于渲染程度对屏幕中的混合区域进行绿到红的高亮（也就是多个半透明图层的叠加），由于重绘的原因，混合对GPU性能会有影响，同时也是滑动或者动画掉帧的罪魁祸首之一

GPU每一帧的绘制的像素有最大限制，这个情况下可以轻易绘制整个屏幕的像素，但如果发生重叠像素的关系需要不停的重绘同一区域的，掉帧和卡顿就有可能发生

GPU会放弃绘制那些完全被其他图层遮挡的像素，但是要计算出一个图层是否被遮挡也是相当复杂并且会消耗CPU的资源，同样，合并不同图层的透明重叠元素消耗的资源也很大，所以，为了快速处理，一般不要使用透明图层

#### 2.3.2 Color Hits Green and Misses Red(光栅化缓存图层的命中情况)

这个选项主要是检测我们有无滥用或正确使用layer的shouldRasterize属性.成功被缓存的layer会标注为绿色,没有成功缓存的会标注为红色。

很多视图Layer由于Shadow、Mask和Gradient等原因渲染很高，因此UIKit提供了API用于缓存这些Layer,self.layer.shouldRasterize = true系统会将这些Layer缓存成Bitmap位图供渲染使用，如果失效时便丢弃这些Bitmap重新生成。图层Rasterization栅格化好处是对刷新率影响较小，坏处是删格化处理后的Bitmap缓存需要占用内存，而且当图层需要缩放时，要对删格化后的Bitmap做额外计算。 使用这个选项后时，如果Rasterized的Layer失效，便会标注为红色，如果有效标注为绿色。当测试的应用频繁闪现出红色标注图层时，表明对图层做的Rasterization作用不大。

#### 2.3.3 Color Copied Image (拷贝的图片)

这个选项主要检查我们有无使用不正确图片色彩格式,由于手机显示都是基于像素的，所以当手机要显示一张图片的时候，系统会帮我们对图片进行转化。比如一个像素占用一个字节，平时我们使用的jpg或者png的图片会进行压缩，但是是可逆的。所以此时，如果图片的格式不正确，则系统将图片转化为像素的时间就有可能变长。而该选项就是检测图片的格式是否是系统所支持的，若是GPU不支持的色彩格式的图片则会标记为青色,则只能由CPU来进行处理。CPU被强制生成了一些图片，然后发送到渲染服务器，而不是简单的指向原始图片的的指针。我们不希望在滚动视图的时候,CPU实时来进行处理,因为有可能会阻塞主线程。iPhone4S以上机型，纹理尺寸上限4096*4096。

#### 2.3.4 Color Immediately (颜色立即更新)

通常 Core Animation 以每秒10次的频率更新图层的调试颜色，对于某些效果来说，这可能太慢了，这个选项可以用来设置每一帧都更新（可能会影响到渲染性能，所以不要一直都设置它）

#### 2.3.5 Color Misaligned Image (图片对齐方式)

这里会高亮那些被缩放或者拉伸以及没有正确对齐到像素边界的图片，即图片Size和imageView中的Size不匹配，会使图过程片缩放，而缩放会占用CPU，所以在写代码的时候保证图片的大小匹配好imageView.被放缩的图片会被标记为黄色，像素不对齐则会标注为紫色。黄色、紫色越多，性能越差

#### 2.3.6 Color Offscreen- Rendered Yellow (离屏渲染)

这里会把那些需要离屏渲染的到图层高亮成黄色。

#### 2.3.7 Color OpenGL Fast Path Blue

这个选项会把任何直接使用 OpenGL 绘制的图层显示为蓝色。蓝色越多，性能越好。如果仅仅使用 UIKit 或者 Core Animation 的 API，那么不会有任何效果。

## 三、实际应用

分别使用了time Profiler 及 view debugging->rending工具对列表页做了分析,查看消耗较大的行为：

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/timeProfiler.png)

此方法内部为富文本属性设置，耗费时间较大。

Color Offscreen- Rendered Yellow（离屏渲染）

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/IMG_0237.PNG)

可以看出离屏渲染只有两处，但并没有对于卡顿问题造成影响。顶部搜索框因为设置了圆角操作，顶部状态栏由于是系统级控件，我对比了其他demo，发现都显示为离屏渲染，应该跟app没有关系，猜测系统内部控件实现问题。

Color Blended Layers（混合过度绘制）

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/blendAfterModify.png)

图为优化前

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/blendBeforeModify.png)

图为优化后。

以下问题并未对卡顿造成实际影响，暂没有更好的处理方式，仅贴出来做参考，未做处理：

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/IMG_71FD7D9691C5-1.jpeg)

Color Copied Image，图中青色表示图片不是GPU直接识别的图片格式，需要CPU预先处理。

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/IMG_0227.PNG)

Color Misaligned Image，此处黄色表示图片尺寸和真实大小不一致，图片被缩放了。紫色表示像素未对齐。

根据分析进行了以下优化：

CPU消耗性操作：

a.文本计算：

车辆名称UILabel富文本属性设置

车辆名称Label由UXINLabel替换，处理富文本设置。删除了无用的图文混排代码。

radarStyleView内车辆名称label改为UXINLabel。

b. 对象创建：

删除了bigImgIcon iconLabel_big控件，这些控件是在双列时展示。

radarStyleView改成懒加载方式，在列表页展示样式时不进行初始化。

c. 约束布局：

删除了多余的高度更新

GPU消耗性操作：

a.Color Blended Layers（混合过度绘制）：

cell中子控件透明度由原clearColor设置为whiteColor，减少图层混合多余的颜色计算。

使用了CADisplayLink测试了列表页的帧率，数据如下

优化前数据：

（y轴为帧率数值，x轴为时间（秒），取了五组数据）

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/listBeforeModify.png)

a. 60 ,47 ,50 ,45 ,54 ,52 ,50 ,53 ,50 ,49 ,52 ,53 ,53 ,48 ,50 ,54 ,39 ,56 ,56 ,58 ,48 ,54 ,52 ,55 ,50 ,54 ,50 ,57 ,52 ,57 ,55

b. 60 ,48 ,58 ,57 ,40 ,56 ,56 ,54 ,53 ,44 ,57 ,51 ,49 ,48 ,54 ,54 ,55 ,48 ,52 ,52 ,50 ,47 ,48 ,59 ,58 ,48 ,53 ,57 ,57 ,51 ,48

c. 60 ,47 ,56 ,52 ,51 ,49 ,54 ,56 ,49 ,58 ,53 ,51 ,57 ,46 ,49 ,49 ,49 ,56 ,50 ,49 ,49 ,56 ,56 ,51 ,54 ,54 ,47 ,57 ,54 ,54 ,48

d. 60 ,49 ,58 ,53 ,56 ,41 ,56 ,55 ,50 ,58 ,56 ,56 ,50 ,54 ,60 ,56 ,51 ,56 ,48 ,53 ,59 ,57 ,59 ,52 ,51 ,57 ,56 ,52 ,56 ,54 ,47

e. 60 ,50 ,59 ,56 ,57 ,55 ,59 ,44 ,59 ,57 ,59 ,52 ,58 ,57 ,56 ,50 ,57 ,58 ,58 ,59 ,55 ,58 ,58 ,49 ,60 ,57 ,59 ,56 ,56 ,48 ,53

平均值分别为：52.0 52.3 52.3 54.1 55.8

总平均值为：53.3

可以看出帧率波动较大，个别会出现较低的帧率。

优化后数据：

![](https://raw.githubusercontent.com/warmShine/MarkDownImageStorage/master/listAfterModify.png)

a. 60 ,51 ,57 ,59 ,59 ,60 ,54 ,59 ,60 ,60 ,58 ,60 ,60 ,55 ,57 ,60 ,60 ,59 ,60 ,60 ,60 ,57 ,57 ,60 ,60 ,60 ,60 ,60 ,59 ,54

b. 60 ,59 ,60 ,53 ,57 ,60 ,60 ,57 ,59 ,59 ,55 ,60 ,59 ,60 ,58 ,57 ,60 ,55 ,60 ,60 ,60 ,60 ,60 ,59 ,56 ,59 ,57 ,60 ,60 ,59

c. 60 ,57 ,60 ,52 ,57 ,59 ,59 ,60 ,59 ,56 ,60 ,59 ,57 ,60 ,60 ,54 ,60 ,59 ,60 ,60 ,57 ,57 ,60 ,60 ,60 ,60 ,60 ,55 ,60 ,59

d. 60 ,50 ,57 ,59 ,59 ,60 ,53 ,60 ,60 ,57 ,60 ,60 ,55 ,59 ,59 ,60 ,59 ,60 ,60 ,55 ,56 ,58 ,60 ,60 ,59 ,54 ,58 ,58 ,60 ,60

e. 60 ,51 ,56 ,59 ,59 ,60 ,52 ,59 ,58 ,59 ,58 ,59 ,60 ,55 ,57 ,60 ,60 ,60 ,60 ,58 ,55 ,60 ,60 ,59 ,60 ,56 ,58 ,60 ,60 ,60

平均值分别为：58.5 58.6 58.5 58.2 58.3

总平均值为：58.4

## 四、总结

对于优化一直有一句话叫：“过早优化是万恶之源”。如果页面没有问题，就没有必

要去做优化，看不到效果。但是当我们要做一个复杂的列表的时候，还是应该考虑一

些可能会造成性能的问题：

4.1 不要创建多余的控件，或者懒加载方式创建控件。

4.2多富文本属性label寻找替代实现方法。

4.3 正确使用图片格式及尺寸

4.4 避免离屏渲染，如圆角、阴影、mask等。

4.5 尽量减少控件alpha通道即多图层混合计算

4.6尽量减少不必要的图层结构

参考文章：

[https://www.jianshu.com/p/86705c95c224](https://www.jianshu.com/p/86705c95c224)

[https://www.cnblogs.com/yecong/p/6501617.html](https://www.cnblogs.com/yecong/p/6501617.html)

[https://www.cnblogs.com/oc-bowen/p/5999997.html](https://www.cnblogs.com/oc-bowen/p/5999997.html)

[https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)