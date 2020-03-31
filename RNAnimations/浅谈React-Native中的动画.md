## 浅谈React-Native中的动画

### React Native 主要的动画分为两大类：
* `LayoutAnimation` 用来实现布局切换的动画。
* `Animated` 更加精确的交互式的动画。

## LayoutAnimation
LayoutAnimation 是在布局发生变化时触发动画的接口，这意味着你需要通过修改布局（比如修改了组件的style、插入新组件）来触发动画。一个常用的调用此API的办法是调用 LayoutAnimation.configureNext，然后调用setState。

### 配置相关:

* `static configureNext(config: Config, onAnimationDidEnd?: Function)` 
计划下一次布局要发生的动画。

* `@param config` 表示动画相应的属性:
  * `duration` 动画持续时间，单位是毫秒。
  * `create` 配置创建新视图时的动画。
  * `update` 配置被更新的视图的动画。

* `@param onAnimationDidEnd` 当动画结束的时候被调用。只在iOS设备上支持。
* `@param onError` 当动画产生错误的时候被调用。只在iOS设备上支持。

* `static create(duration: number, type, creationProp)`
用来创建 configureNext 所需的 config 参数的辅助函数。

### config 参数格式参考：
```
LayoutAnimation.configureNext({
    //动画持续时间
    duration: 700, 
    //布局创建时的动画
    create: { 
    //线性模式，LayoutAnimation.Types.linear 写法亦可   
        type: 'linear', 
    //动画属性，除了opacity 还有一个 scaleXY 可以配置置,LayoutAnimation.Properties.opacity 写法亦可 
        property: 'opacity'  
    },
    //布局更新时的动画
    update: {  
    //弹跳模式
        type: 'spring', 
    //弹跳阻尼系数  
        springDamping: 0.4  
    }
 });
```
### create、update 的属性：
* `duration`：动画持续时间（单位：毫秒）,会覆盖 `config` 中设置的 `duration`。

* `delay`：延迟指定时间（单位：毫秒）。

* `springDamping`：弹跳动画阻尼系数（配合 `spring` 使用）。

* `initialVelocity`：初始速度。

* `type`：类型定义在`LayoutAnimation.Types`中：
   * `spring`：弹跳
   * `linear`：线性
   * `easeInEaseOut`：缓入缓出
   * `easeIn`：缓入
   * `easeOut`：缓出
   
* `property`：类型定义在`LayoutAnimation.Properties`中：
   * `opacity`：透明度
   * `scaleXY`：缩放

### 平移+大小变化示例：
```
import React from 'react';
import styled from 'styled-components/native';
import {
  AppRegistry,
  Text,
  View,
  TouchableHighlight,
  Dimensions,
  LayoutAnimation
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
`
var screenW = Dimensions.get('window').width;
var screenH = Dimensions.get('window').height;

class ReactNativeAnimaDemo extends React.Component {
  constructor(props) {
    super(props)
    this.state = {
      width: 100,
      height: 100,
      left: 20,
      top: 84,
    }
  }
  _startAnimation = () => {
    LayoutAnimation.configureNext({
      duration: 1000,
      create: {
        type: LayoutAnimation.Types.spring,
        property: LayoutAnimation.Properties.opacity
      },
      update: {
        type: 'spring',
        springDamping: 0.4,
      }
    });
    this.setState({
      width: this.state.width + 40,
      height: this.state.height + 60,
      left: this.state.left + 20,
      top: this.state.top + 50,
    });
  }

  render() {
    return (
      <ContentView >
        <View 
        style = {{width:this.state.width,height:this.state.height,position:'absolute',left:this.state.left, top:this.state.top}}
        backgroundColor = 'red'/>
        <TouchableHighlight 
        style = {{position:'absolute',width:screenW,height:50,top:screenH-80,backgroundColor:'white',alignItems:'center'}} 
        onPress={()=>this._startAnimation()}>
          <Text 
          style={{width:200,height:50,textAlign:'center',lineHeight:50}}
          >
          点击开始动画
          </Text>
          </TouchableHighlight>
      </ContentView>

    );
  }
}

// 整体js模块的名称
AppRegistry.registerComponent('ReactNativeAnimaDemo', () => ReactNativeAnimaDemo);
```

### 视图创建动画示例：
```
import React, { Component } from 'react';
import styled from 'styled-components/native';
import {
    Text,
    View,
    TouchableOpacity,
    LayoutAnimation,
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
alignItems:center;
justifyContent:center

`
var anima = {
    duration: 1000,   //持续时间
    create: {
        type: LayoutAnimation.Types.linear,
        property: LayoutAnimation.Properties.scaleXY,
    },
    update: {
        type: 'linear',
    }
};

export default class ScaleAnima extends React.Component {
    constructor(props) {
        super(props)

        this.state = {
            width: 250,
            height: 125,
            show:false,
        }
    }

    _clickStartAnimation() {
        LayoutAnimation.configureNext(anima,()=>{});
        this.setState({
            show: true,
        });
    }

    render(){

        var secondView = this.state.show ? (
            <View style={{width:this.state.width,height:this.state.height,backgroundColor:'yellow'}}>

            </View>
        ) : null;

        return (
            <ContentView>

                <View style={{width:this.state.width,height:this.state.height,backgroundColor:'red'}}
                       >
                </View>

                {secondView}

                <TouchableOpacity style={{width:200,height:50,backgroundColor:'yellow',marginTop:40}} onPress={this._clickStartAnimation.bind(this)}>
                    <Text style={{width:200,height:50,textAlign:'center',lineHeight:50}}>视图创建</Text>
                </TouchableOpacity>
            </ContentView>
        );
    }
}
```
### 实现拆解：
首先创建一个 config 变量，1秒的持续时间，type 为 linear，property 为
scaleXY，然后在点击按钮之后，修改 state.show 为 true，然后根据这个属性就会加载出一个新的View，因为是创建，不是更新，所以新的View是执行的 create 内的动画效果，原来的View和按钮则是执行的 Update 效果。

### LayoutAnimation.create:
如果觉得上面的代码太繁杂，还可以通过 LayoutAnimation.create 这个函数更简单的创建一个config，也可以实现上面的动画效果。

* `create`函数接受三个参数：

 * `duration`：动画持续时间。
 
 * `type`：`create` 和 `update` 时的动画类型，定义在 `LayoutAnimation.Types`。
 
 * `creationProp`：`create` 时的动画属性，定义 `LayoutAnimation.Properties`。
 
 ```
...
_clickStartAnimation() {
    LayoutAnimation.configureNext(LayoutAnimation.create(1000,
        LayoutAnimation.Types.linear,
        LayoutAnimation.Properties.scaleXY));
    this.setState({
        show: true,
    });
}
...
```
### LayoutAnimation.Presets

系统也为我们提供了3个默认的动画，定义在LayoutAnimation.Presets中：

* easeInEaseOut：缓入缓出
* linear：线性
* spring：弹性
可以使用这种方式创建一些简单的动画效果。

```
...
_clickStartAnimation() {
    LayoutAnimation.configureNext(LayoutAnimation.Presets.linear);                
    this.setState({
        show: true,    
    });
}
...
```
这个效果和上面类似，区别在于这个默认 Properties 是 opacity，创建过程是透明度的变化。

## Animated
### 简介
Animated 库用于创建更精细的交互控制的动画，它使得开发者可以非常容易地实现各种各样的动画和交互方式，并且具备极高的性能。Animated 旨在以声明的形式来定义动画的输入与输出，在其中建立一个可配置的变化函数，然后使用简单的 start/stop 方法来控制动画按顺序执行。

### `Animated` 提供了两种类型的值：

* `Animated.Value()` 用于单个值。

* `Animated.ValueXY()` 用于矢量值。

Animated.Value() 可以绑定到样式或是其他属性上，也可以进行插值运算。单个 Animated.Value() 可以用在任意多个属性上。

### `Animated` 用于创建动画的主要方法：


* `Animated.timing()`：最常用的动画类型，使一个值按照一个过渡曲线而随时间变化。

* `Animated.spring()`：弹簧效果，基础的单次弹跳物理模型实现的 spring 动画。

* `Animated.decay()`：衰变效果，以一个初始的速度和一个衰减系数逐渐减慢变为0。

### `Animated` 实现组合动画的主要方式：


* `Animated.parallel()`：同时开始一个动画数组里的全部动画。默认情况下，如果有任何一个动画停止了，其余的也会被停止。可以通过`stopTogether` 选项设置为 false 来取消这种关联。

* `Animated.sequence()`：按顺序执行一个动画数组里的动画，等待一个完成后再执行下一个。如果当前的动画被中止，后面的动画则不会继续执行。

* `Animated.stagger()`：一个动画数组，传入一个时间参数来设置队列动画间的延迟，即在前一个动画开始之后，隔一段指定时间才开始执行下一个动画里面的动画，并不关心前一个动画是否已经完成，所以有可能会出现同时执行（重叠）的情况。

### 插值函数：

* `interpolate()`：将输入值范围转换为输出值范围。
譬如：把0-1映射到0-10。

接下来，我们结合一个个的例子介绍它们的用法：

### 1. `Animated.timing()`
最常用的动画类型，使一个值按照一个过渡曲线而随时间变化，格式如下：
`Animated.timing(animateValue, conf<Object>)`，conf参数格式：

```
{
  duration：动画持续的时间（单位是毫秒），默认为500。
  easing：一个用于定义曲线的渐变函数。Easing模预置了 linear、ease、elastic、bezier 等诸多缓动特性。iOS默认为Easing.inOut(Easing.ease)，。
  delay：开始动画前的延迟时间（毫秒），默认为0。
}

```
### 示例
```
import React from 'react';
import styled from 'styled-components/native';
import {
  Text,
  View,
  TouchableOpacity,
  Animated,
  Easing,
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
alignItems:center;
justifyContent:center
`


export default class AnimaViewOpacity extends React.Component {
    constructor(props) {
        super(props)
        this.state = {
            fadeOutOpacity: new Animated.Value(1),
        }

        this.fadeOutAnimated = Animated.timing(
            this.state.fadeOutOpacity,
            {
                toValue: 0,  //透明度动画最终值
                duration: 3000,   //动画时长3000毫秒
                easing: Easing.linear,
            }
        );
    }

    _startAnimated() {
        this.fadeOutAnimated.start(() => this.state.fadeOutOpacity.setValue(1));
    }

    render(){
        return (
            <ContentView>
                 <Animated.View style={{width: 200, height: 300, opacity: this.state.fadeOutOpacity}}>
                    <View  style={{width:200,height:300,backgroundColor:'white'}}
                           >
                    </View>
                 </Animated.View>

                <TouchableOpacity style={{width:200,height:50,backgroundColor:'yellow',marginTop:40}} onPress={this._startAnimated.bind(this)}>
                    <Text style={{width:200,height:50,textAlign:'center',lineHeight:50}}>点击开始动画</Text>
                </TouchableOpacity>
            </ContentView>
        );
    }
}
```
上面的代码做了下面这些事情：

* 1.首先实例化动画的初始值 `fadeOutOpacity`，作为图片的透明度值。
* 2.然后定义了一个 `Animated.timing()` 动画事件，并配置相应的参数，把它赋予 `this.fadeOutAnimated` 变量，在后续可以使用 `.start()` 或 `.stop()` 方法来开始/停止该动画。
* 3.把动画添加到在 `<Animate.View></Animate.View>` 上，我们把实例化的动画初始值传入 `style` 中的 `opacity`。
* 4.最后我们点击按钮，调用 `this.fadeOutAnimated.start()` 方法开启动画，在这里将 () => `this.state.fadeOutOpacity.setValue(1)` 传给了 `start()`，用于在一次动画完成之后把图片的透明度再次设置为1，这也是创建无穷动画的方式。

### 2.`interpolate()`  实现映射

### 示例

```
import React from 'react';
import styled from 'styled-components/native';
import {
  Text,
  TouchableOpacity,
  Animated,
  Easing,
  Image,
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
alignItems:center;
justifyContent:center
`


export default class Mixture extends React.Component {
    constructor(props) {
        super(props)
        this.state = {
            animatedValue: new Animated.Value(0),
        }
        this.rotateAnimated = Animated.timing(
            this.state.animatedValue,
            {
                toValue: 1,
                duration: 3000,
                easing: Easing.in,
            }
        );
    }

    _startAnimated() {
        this.state.animatedValue.setValue(0);
        this.rotateAnimated.start(() => this._startAnimated());
    }

    render(){
        const rotateZ = this.state.animatedValue.interpolate({
            inputRange: [0, 1],
            outputRange: ['0deg', '360deg']
        });

        const opacity = this.state.animatedValue.interpolate({
            inputRange: [0, 0.5, 1],
            outputRange: [0, 1, 0]
        });

        const rotateX = this.state.animatedValue.interpolate({
            inputRange: [0, 0.5, 1],
            outputRange: ['0deg', '180deg', '0deg']
        });

        const textSize = this.state.animatedValue.interpolate({
            inputRange: [0, 0.5, 1],
            outputRange: [18, 32, 18]
        });

        const marginLeft = this.state.animatedValue.interpolate({
            inputRange: [0, 0.5, 1],
            outputRange: [0, 200, 0]
        });

        return (
            <ContentView>

                <Animated.View
                    style={{
                        marginTop: 10,
                        width: 100,
                        height: 100,
                        transform: [
                            {rotateZ:rotateZ},
                        ]
                    }}
                >
                    <Image style={{width:100,height:100}}
                           source={require('../source/out_loading_image.png')}>
                    </Image>
                </Animated.View>

                <Animated.View
                    style={{
                        marginTop: 10,
                        width: 100,
                        height: 100,
                        opacity:opacity,
                        backgroundColor:'red',
                    }}
                />

                <Animated.Text
                    style={{
                        marginTop: 10,
                        width:100,
                        fontSize: 18,
                        color: 'white',
                        backgroundColor:'red',
                        transform: [
                            {rotateX:rotateX},
                        ]
                    }}
                >
                    翻转动画。
                </Animated.Text>

                <Animated.Text
                    style={{
                        marginTop: 10,
                        height: 100,
                        lineHeight: 100,
                        fontSize: textSize,
                        color: 'red'
                    }}
                >
                    缩放动画
                </Animated.Text>

                <Animated.View
                    style={{
                        marginTop: 10,
                        width: 100,
                        height: 100,
                        marginLeft:marginLeft,
                        backgroundColor:'red',
                    }}
                />

                <TouchableOpacity style={{width:200,height:50,backgroundColor:'yellow',marginTop:40}} onPress={this._startAnimated.bind(this)}>
                    <Text style={{width:200,height:50,textAlign:'center',lineHeight:50}}>点击开始动画</Text>
                </TouchableOpacity>
            </ContentView>
        );
    }
}

```
其中 transform 是一个变换数组，常用的有 scale, scaleX, scaleY, translateX, translateY, rotate, rotateX, rotateY, rotateZ，其他用法与上面相同。

### 3. `Animated.spring()`

弹簧效果，基础的单次弹跳物理模型实现的 spring 动画，格式如下：
`Animated.spring(animateValue, conf<Object>)`，conf参数格式：

```
{
  friction: 控制“弹跳系数”、夸张系数，默认为7。
  tension: 控制速度，默认为40。
  speed: 控制动画的速度，默认为12。
  bounciness: 反弹系数，默认为8。
}
```
### 示例 

```
import React from 'react';
import styled from 'styled-components/native';
import {
  Text,
  TouchableOpacity,
  Animated,
  View,
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
alignItems:center;
justifyContent:center
`

export default class AnimatedSpring extends React.Component {

    constructor(props) {
        super(props);

        this.state = {
            springValue: new Animated.Value(1),
        };

        this.springAnimated = Animated.spring(
            this.state.springValue,
            {
                toValue: 1,
                friction: 1,    //弹跳系数
                tension: 1,   // 控制速度
                // bounciness:50,
            }
        );
    }

    _startAnimated() {
        this.state.springValue.setValue(0.1);
        this.springAnimated.start();
    }

    render(){
        return (
            <ContentView>

                <Animated.View
                    style={{
                        width: 282,
                        height: 51,
                        transform:[
                            {scale:this.state.springValue}
                        ]
                    }}
                >
                    <View style={{width:282,height:51,backgroundColor:'white'}}
                           >
                    </View>
                </Animated.View>

                <TouchableOpacity style={{width:200,height:50,backgroundColor:'yellow',marginTop:40}} onPress={this._startAnimated.bind(this)}>
                    <Text style={{width:200,height:50,textAlign:'center',lineHeight:50}}>点击开始动画</Text>
                </TouchableOpacity>
            </ContentView>
        );
    }
}
```
上面的代码做了下面这些事情：

1.首先实例化动画的初始值 `springValue`，作为图片的初始 scale 大小。
2.然后定义了一个 `Animated. spring()` 动画事件，并配置相应的参数，把它赋予 `this. springAnimated `变量。
3.把动画添加到在` <Animate.View></Animate.View>` 上，我们把实例化的动画初始值传入 style 中的 `transform:[{scale:this.state.springValue}]。`
4.最后我们点击按钮，在动画开启之前把 scale 调到0.1，然后调用 `this. springAnimated.start()` 方法开启动画。


### 4. `Animated.decay()`

衰变效果，以一个初始的速度和一个衰减系数逐渐减慢变为0，格式如下：
`Animated.decay(animateValue, conf<Object>)`，conf参数格式：

```
{
  velocity: 起始速度，必填参数。
  deceleration: 速度衰减比例，默认为0.997。
}
```
### 示例
```
import React from 'react';
import styled from 'styled-components/native';
import {
  Text,
  TouchableOpacity,
  Animated,
  View,
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
alignItems:center;
justifyContent:center
`
export default class AnimatedDecay extends React.Component {

    constructor(props) {
        super(props);

        this.state = {
            decayValue: new Animated.ValueXY({x:0,y:0}),
        };

        this.decayAnimated = Animated.decay(
            this.state.decayValue,
            {
                velocity: 50,       // 起始速度，必填
                deceleration: 0.2,  // 速度衰减比例，默认为0.997
            }
        );
    }

    _startAnimated() {
        this.decayAnimated.start();
    }

    render(){
        return (
            <ContentView>

                <Animated.View
                    style={{
                        width: 100,
                        height: 150,
                        transform:[
                            {translateX: this.state.decayValue.x}, // x轴移动
                            {translateY: this.state.decayValue.y}, // y轴移动
                        ]
                    }}
                >
                    <View style={{width:100,height:150,backgroundColor:'white'}}
                           >
                    </View>
                </Animated.View>

                <TouchableOpacity style={{width:200,height:50,backgroundColor:'yellow',marginTop:200}} onPress={this._startAnimated.bind(this)}>
                    <Text style={{width:200,height:50,textAlign:'center',lineHeight:50}}>点击开始动画</Text>
                </TouchableOpacity>
            </ContentView>
        );
    }
}

```
以上就是所有的单个动画执行方式，接下来我们来看组合动画。

### 5. `Animated.parallel()`

同时开始一个动画数组里的全部动画。默认情况下，如果有任何一个动画停止了，其余的也会被停止。可以通过`stopTogether` 选项设置为 false 来取消这种关联，格式如下：
`Animated.parallel(Animates<Array>, [conf<Object>])：`，conf参数格式：

```
{
  stopTogether: false
}
```
### 示例

```

import React from 'react';
import styled from 'styled-components/native';
import {
  Text,
  TouchableOpacity,
  Animated,
  View,
  Image,
  Easing
} from 'react-native';

const ContentView = styled.View`
flex:1
backgroundColor:black
alignItems:center;
justifyContent:center
`
export default class AnimatedParallel extends React.Component {

    constructor(props) {
        super(props);

        this.state = {
            dogOpacityValue: new Animated.Value(1),
            dogACCValue : new Animated.Value(0)
        };

        this.parallelAnimated = Animated.parallel(
            [
                Animated.timing(
                    this.state.dogOpacityValue,
                    {
                        toValue: 1,
                        duration: 1000,
                    }
                ),
                Animated.timing(
                    this.state.dogACCValue,
                    {
                        toValue: 1,
                        duration: 2000,
                        easing: Easing.linear,
                    }
                ),
            ],
            {
                stopTogether: false
            }
        );
    }

    _startAnimated() {
        this.state.dogOpacityValue.setValue(0);
        this.state.dogACCValue.setValue(0);
        this.parallelAnimated.start();
    }

    render(){

        //透明度
        const dogOpacity = this.state.dogOpacityValue.interpolate({
            inputRange: [0,0.2,0.4,0.6,0.8,1],
            outputRange: [0,1,0,1,0,1]
        });

        //项链上面
        const neckTop = this.state.dogACCValue.interpolate({
            inputRange: [0, 1],
            outputRange: [350, 235]
        });

        //眼镜左边
        const left = this.state.dogACCValue.interpolate({
            inputRange: [0, 1],
            outputRange: [-120, 127]
        });

        //眼镜旋转
        const rotateZ = this.state.dogACCValue.interpolate({
            inputRange: [0, 1],
            outputRange: ['0deg', '360deg']
        });

        return (
            <ContentView>

                {/*// 狗头*/}
                <Animated.View
                    style={{
                        width: 375,
                        height: 240,
                        opacity:dogOpacity,
                    }}
                >
                    <Image style={{width:375,height:242}}
                           source={require("../source/dog.jpg")}>
                    </Image>
                </Animated.View>

                {/*// 项链*/}
                <Animated.View
                    style={{
                        width: 250,
                        height: 100,
                        position: 'absolute',
                        top:neckTop,
                        left:93,
                    }}
                >
                    <Image style={{width:250,height:100,resizeMode:'stretch'}}
                           source={require('../source/necklace.jpg')}>
                    </Image>
                </Animated.View>

                <View
                    style={{
                        width: 375,
                        height: 200,
                        backgroundColor:'white',
                    }}
                />

                {/*// 眼镜*/}
                <Animated.View
                    style={{
                        width: 120,
                        height: 25,
                        position: 'absolute',
                        top:165,
                        left:left,
                        transform:[
                            {rotateZ:rotateZ}
                        ],
                    }}
                >
                    <Image  style={{width:120,height:25,resizeMode:'stretch'}}
                           source={require('../source/glasses.png')}>
                    </Image>
                </Animated.View>

                <TouchableOpacity style={{width:375,height:20,backgroundColor:'white',marginTop:150}} onPress={this._startAnimated.bind(this)}>
                    <Text style={{width:375,height:20,textAlign:'center',lineHeight:20}}>点击开始动画</Text>
                </TouchableOpacity>
            </ContentView>
        );
    }
}
```

* 1.首先实例化动画的初始值 dogOpacityValue，作为狗头图片的初始透明度，实例化动画的初始值 dogACCValue，作为墨镜、项链图片的初始值。
* 2.然后定义了两个 Animated. timing() 动画事件，并配置相应的参数，第一个是对应狗头图片的透明度，第二个对应的是墨镜、金链子的 timing 动画，然后把这两个动画事件放入到 Animated.parallel() 的动画数组内，最后把 Animated.parallel() 赋予 this.parallelAnimated 变量，用法其实和单独的动画事件差不多。
* 3.在 render 方法中，创建了 dogOpacity、left、rotateZ、neckTop 等变量，并调用了 interpolate() 方法对它们一一映射，分别对应的是透明度、金链子 top 的距离、墨镜 left 距离、眼镜旋转的角度，结合上面代码查看具体信息。
* 4.把动画添加到在不同的 <Animate.View></Animate.View> 上，我们把实例化的动画初始值传入 style 中。
* 5.最后动画开启之前把 dogOpacityValue、dogACCValue 调到0，然后调用 this. parallelAnimated.start() 方法开启动画。


### 6.`Animated.sequence()`
按顺序执行一个动画数组里的动画，等待一个完成后再执行下一个。如果当前的动画被中止，后面的动画则不会继续执行，格式如下：
`Animated.sequence(Animates<Array>)`

### 7. `Animated.stagger()`
一个动画数组，传入一个时间参数来设置队列动画间的延迟，即在前一个动画开始之后，隔一段指定时间才开始执行下一个动画里面的动画，并不关心前一个动画是否已经完成，所以有可能会出现同时执行（重叠）的情况，其格式如下：
Animated.stagger(delayTime<Number>, Animates<Array>)
其中 delayTime 为指定的延迟时间（毫秒），第二个和上面两个一样传入一个动画事件数组。