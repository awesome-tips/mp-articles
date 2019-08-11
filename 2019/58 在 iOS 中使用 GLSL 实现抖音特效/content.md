本文通过模仿抖音中几种特效的实现，来讲解 GLSL 的实际应用。

## 前言

本文的灵感来自于 《当一个 Android 开发玩抖音玩疯了之后(二)》<sup>[1]</sup> 这篇文章。

这位博主在 Android 平台上，通过自己的分析，尝试还原了抖音上的几种视频特效。他是通过「部分 GLSL 代码 + 部分 Java 代码」的方式来实现的。

读完之后，在膜拜之余，我产生了一个大胆的想法：**我可不可以在 iOS 上，只通过纯 GLSL 的编写，来实现类似的效果呢？**

很好的想法，不过，由于抖音的特效是基于视频的滤镜，我们在这之前只讲到了关于图片的渲染，如果马上跳跃到视频的部分，好像有点超纲了。

于是，我又有了一个更大胆的想法：**我可不可以在 iOS 上，只通过纯 GLSL 的编写，在静态的图片上，实现类似的效果呢？**

这样的话，我们就可以把更多的注意力放在 GLSL 本身，而不是视频的采集和输出上面。

于是，就有了这篇文章。为了无缝地过渡，我会沿用之前 `GLSL 渲染的例子`<sup>[2]</sup> ，只改变 Shader 部分的代码，来尝试还原那篇文章中实现的六种特效。

## 〇、动画

你可能会问：抖音上的特效都是动态的，要怎么把动态的效果，加到一个静态的图片上呢？

问的好，所以第一步，我们就要让静态的图片动起来。

回想一下，我们在 UIKit 中实现的动画，无非就是把指令发送给 CoreAnimation，然后在屏幕刷新的时候，CoreAnimation 会去逐帧计算当前应该显示的图像。

这里的重点是「逐帧计算」。在 OpenGL ES 中也是类似，我们实现动画的方式，就是自己去计算每一帧应该显示的图像，然后在屏幕刷新的时候，重新渲染。

这个「逐帧计算」的过程，我们是放到 Shader 中进行的。然后我们可以通过一个表示时间的参数，在重新渲染的时候，传入当前的时间，让 Shader 计算出当前动画的进度。至于重新渲染的时机，则是依靠 CADisplayLink 来实现的。

具体代码大概像这样：

```objc
self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(timeAction)];
[self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];

// 
- (void)timeAction {
    glUseProgram(self.program);
    glBindBuffer(GL_ARRAY_BUFFER, self.vertexBuffer);
    
    // 传入时间
    CGFloat currentTime = self.displayLink.timestamp - self.startTimeInterval;
    GLuint time = glGetUniformLocation(self.program, "Time");
    glUniform1f(time, currentTime);

    // 清除画布
    glClear(GL_COLOR_BUFFER_BIT);
    glClearColor(1, 1, 1, 1);
    
    // 重绘
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    [self.context presentRenderbuffer:GL_RENDERBUFFER];
}
```
相应地，在 Shader 中有一个 uniform 修饰的 Time 参数：

```objc
uniform float Time;
```

这样 Shader 就可以通过 Time 来计算出当前应该显示的图像了。

## 一、缩放

### 1、最终效果

![1](http://)

我们要实现的第一种效果是「缩放」，看起来很简单，可以通过修改顶点坐标和纹理坐标的对应关系来实现。

这是一个很基础的效果，在下面的其它特效中还会用到。修改坐标的对应关系可以通过修改顶点着色器，或者修改片段着色器来实现。 这里先讲修改顶点着色器的方式，在后面的特效中会再提一下修改片段着色器的方式。

### 2、代码实现

顶点着色器代码：

```c
attribute vec4 Position;
attribute vec2 TextureCoords;
varying vec2 TextureCoordsVarying;

uniform float Time;

const float PI = 3.1415926;

void main (void) {
    float duration = 0.6;
    float maxAmplitude = 0.3;
    
    float time = mod(Time, duration);
    float amplitude = 1.0 + maxAmplitude * abs(sin(time * (PI / duration)));
    
    gl_Position = vec4(Position.x * amplitude, Position.y * amplitude, Position.zw);
    TextureCoordsVarying = TextureCoords;
}
```

这里的 duration 表示一次缩放周期的时长，`mod(Time, duration)` 表示将传入的时间转换到一个周期内，即 time 的范围是 0 ~ 0.6，amplitude 表示振幅，引入 PI 的目的是为了使用 sin 函数，将 amplitude 的范围控制在 1.0 ~ 1.3 之间，并随着时间变化。

这里放大的关键在于 `vec4(Position.x * amplitude, Position.y * amplitude, Position.zw)` ，我们将顶点坐标的 x 和 y 分别乘上一个放大系数，在纹理坐标不变的情况下，就达到了拉伸的效果。

## 二、灵魂出窍

### 1、最终效果

![2](http://)

「灵魂出窍」看上去是两个层的叠加，并且上面的那层随着时间的推移，会逐渐放大且不透明度逐渐降低。这里也用到了放大的效果，我们这次用片段着色器来实现。

### 2、代码实现

片段着色器代码：

```c
precision highp float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

uniform float Time;

void main (void) {
    float duration = 0.7;
    float maxAlpha = 0.4;
    float maxScale = 1.8;
    
    float progress = mod(Time, duration) / duration; // 0~1
    float alpha = maxAlpha * (1.0 - progress);
    float scale = 1.0 + (maxScale - 1.0) * progress;
    
    float weakX = 0.5 + (TextureCoordsVarying.x - 0.5) / scale;
    float weakY = 0.5 + (TextureCoordsVarying.y - 0.5) / scale;
    vec2 weakTextureCoords = vec2(weakX, weakY);
    
    vec4 weakMask = texture2D(Texture, weakTextureCoords);
    
    vec4 mask = texture2D(Texture, TextureCoordsVarying);
    
    gl_FragColor = mask * (1.0 - alpha) + weakMask * alpha;
}
```

首先是放大的效果。关键点在于 weakX 和 weakY 的计算，比如 `0.5 + (TextureCoordsVarying.x - 0.5) / scale` 这一句的意思是，将顶点坐标对应的纹理坐标的 x 值到纹理中点的距离，缩小一定的比例。这次我们是改变了纹理坐标，而保持顶点坐标不变，同样达到了拉伸的效果。

然后是两层叠加的效果。通过上面的计算，我们得到了两个纹理颜色值 weakMask 和 mask， weakMask 是在 mask 的基础上做了放大处理。

我们将两个颜色值进行叠加需要用到一个公式：**最终色 = 基色 * a% + 混合色 * (1 - a%)** ，这个公式来自 混合模式中的正常模式<sup>[3]</sup> 。

这个公式表明了一个不透明的层和一个半透明的层进行叠加，重叠部分的最终颜色值。因此，上面叠加的最终结果是 **mask * (1.0 - alpha) + weakMask * alpha** 。

## 三、抖动

### 1、最终效果

![3](http://)

「抖动」是很经典的抖音的颜色偏移效果，其实这个效果实现起来还挺简单的。另外，除了颜色偏移，可以看到还有微弱的放大效果。

2、代码实现

片段着色器代码：

```c
precision highp float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

uniform float Time;

void main (void) {
    float duration = 0.7;
    float maxScale = 1.1;
    float offset = 0.02;
    
    float progress = mod(Time, duration) / duration; // 0~1
    vec2 offsetCoords = vec2(offset, offset) * progress;
    float scale = 1.0 + (maxScale - 1.0) * progress;
    
    vec2 ScaleTextureCoords = vec2(0.5, 0.5) + (TextureCoordsVarying - vec2(0.5, 0.5)) / scale;
    
    vec4 maskR = texture2D(Texture, ScaleTextureCoords + offsetCoords);
    vec4 maskB = texture2D(Texture, ScaleTextureCoords - offsetCoords);
    vec4 mask = texture2D(Texture, ScaleTextureCoords);
    
    gl_FragColor = vec4(maskR.r, mask.g, maskB.b, mask.a);
}
```

这里的放大和上面类似，我们主要看一下颜色偏移。颜色偏移是对三个颜色通道进行分离，并且给红色通道和蓝色通道添加了不同的位置偏移，代码很容易看懂。

## 四、闪白

### 1、最终效果

![4](http://)

「闪白」其实看起来一点儿也不酷炫，而且看久了还容易被闪瞎。这个效果实现起来也十分简单，无非就是叠加一个白色层，然后白色层的透明度随着时间不断地变化。

### 2、代码实现

片段着色器代码：

```c
precision highp float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

uniform float Time;

const float PI = 3.1415926;

void main (void) {
    float duration = 0.6;
    
    float time = mod(Time, duration);
    
    vec4 whiteMask = vec4(1.0, 1.0, 1.0, 1.0);
    float amplitude = abs(sin(time * (PI / duration)));
    
    vec4 mask = texture2D(Texture, TextureCoordsVarying);
    
    gl_FragColor = mask * (1.0 - amplitude) + whiteMask * amplitude;
}
```

在上面「灵魂出窍」的例子中，我们已经知道了如何实现两个层的叠加。这里我们只需要创建一个白色的层 whiteMask，然后根据当前的透明度来计算最终的颜色值即可。

## 五、毛刺

### 1、最终效果

![5](http://)

终于有了一个稍微复杂一点的效果，「毛刺」看上去是「撕裂 + 微弱的颜色偏移」。颜色偏移我们在上面已经实现，这里主要是讲解撕裂的效果。

具体的思路是，我们让每一行像素随机偏移 -1 ~ 1 的距离（这里的 -1 ~ 1 是对于纹理坐标来说的），但是如果整个画面都偏移比较大的值，那我们可能都看不出原来图像的样子。所以我们的逻辑是，**设定一个阈值，小于这个阈值才进行偏移，超过这个阈值则乘上一个缩小系数**。

则最终呈现的效果是：**绝大部分的行都会进行微小的偏移，只有少量的行会进行较大偏移**。

### 2、代码实现

片段着色器代码：

```c
precision highp float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

uniform float Time;

const float PI = 3.1415926;

float rand(float n) {
    return fract(sin(n) * 43758.5453123);
}

void main (void) {
    float maxJitter = 0.06;
    float duration = 0.3;
    float colorROffset = 0.01;
    float colorBOffset = -0.025;
    
    float time = mod(Time, duration * 2.0);
    float amplitude = max(sin(time * (PI / duration)), 0.0);
    
    float jitter = rand(TextureCoordsVarying.y) * 2.0 - 1.0; // -1~1
    bool needOffset = abs(jitter) < maxJitter * amplitude;
    
    float textureX = TextureCoordsVarying.x + (needOffset ? jitter : (jitter * amplitude * 0.006));
    vec2 textureCoords = vec2(textureX, TextureCoordsVarying.y);
    
    vec4 mask = texture2D(Texture, textureCoords);
    vec4 maskR = texture2D(Texture, textureCoords + vec2(colorROffset * amplitude, 0.0));
    vec4 maskB = texture2D(Texture, textureCoords + vec2(colorBOffset * amplitude, 0.0));
    
    gl_FragColor = vec4(maskR.r, mask.g, maskB.b, mask.a);
}
```

上面提到的像素随机偏移需要用到随机数，可惜 GLSL 里并没有内置的随机函数，所以我们需要自己实现一个。

这个 **float rand(float n)** 的实现看上去很神奇，它其实是来自 这里<sup>[4]</sup> ，江湖人称「噪声函数」。

它其实是一个伪随机函数，本质上是一个 Hash 函数。但在这里我们可以把它当成随机函数来使用，它的返回值范围是 0 ~ 1。如果你对这个函数想了解更多的话可以看 这里<sup>[5]</sup> 。

## 六、幻觉

### 1、最终效果

![6](http://)

「幻觉」这个效果有点一言难尽，因为其实看上去并不是很像。原来的效果是基于视频上一帧的结果去合成，静态的图片很难模拟出这种情况。不管怎么说，既然已经尽力，不像就不像吧，下面讲一下我的实现思路。

可以看出这个效果是**残影和颜色偏移**的叠加。

残影的效果还好，在移动的过程中，每经过一段时间间隔，根据当前的位置去创建一个新层，并且新层的不透明度随着时间逐渐减弱。于是在一个移动周期内，可以看到很多透明度不同的层叠加在一起，从而形成残影的效果。

然后是这个颜色偏移。我们可以看到，物体移动的过程是蓝色在前面，红色在后面。所以整个过程可以理解成：在移动的过程中，每间隔一段时间，遗失了一部分红色通道的值在原来的位置，并且这部分红色通道的值，随着时间偏移，会逐渐恢复。

### 2、代码实现

片段着色器代码：

```c
precision highp float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

uniform float Time;

const float PI = 3.1415926;
const float duration = 2.0;

vec4 getMask(float time, vec2 textureCoords, float padding) {
    vec2 translation = vec2(sin(time * (PI * 2.0 / duration)),
                            cos(time * (PI * 2.0 / duration)));
    vec2 translationTextureCoords = textureCoords + padding * translation;
    vec4 mask = texture2D(Texture, translationTextureCoords);
    
    return mask;
}

float maskAlphaProgress(float currentTime, float hideTime, float startTime) {
    float time = mod(duration + currentTime - startTime, duration);
    return min(time, hideTime);
}

void main (void) {
    float time = mod(Time, duration);
    
    float scale = 1.2;
    float padding = 0.5 * (1.0 - 1.0 / scale);
    vec2 textureCoords = vec2(0.5, 0.5) + (TextureCoordsVarying - vec2(0.5, 0.5)) / scale;
    
    float hideTime = 0.9;
    float timeGap = 0.2;
    
    float maxAlphaR = 0.5; // max R
    float maxAlphaG = 0.05; // max G
    float maxAlphaB = 0.05; // max B
    
    vec4 mask = getMask(time, textureCoords, padding);
    float alphaR = 1.0; // R
    float alphaG = 1.0; // G
    float alphaB = 1.0; // B
    
    vec4 resultMask;
    
    for (float f = 0.0; f < duration; f += timeGap) {
        float tmpTime = f;
        vec4 tmpMask = getMask(tmpTime, textureCoords, padding);
        float tmpAlphaR = maxAlphaR - maxAlphaR * maskAlphaProgress(time, hideTime, tmpTime) / hideTime;
        float tmpAlphaG = maxAlphaG - maxAlphaG * maskAlphaProgress(time, hideTime, tmpTime) / hideTime;
        float tmpAlphaB = maxAlphaB - maxAlphaB * maskAlphaProgress(time, hideTime, tmpTime) / hideTime;
     
        resultMask += vec4(tmpMask.r * tmpAlphaR,
                           tmpMask.g * tmpAlphaG,
                           tmpMask.b * tmpAlphaB,
                           1.0);
        alphaR -= tmpAlphaR;
        alphaG -= tmpAlphaG;
        alphaB -= tmpAlphaB;
    }
    resultMask += vec4(mask.r * alphaR, mask.g * alphaG, mask.b * alphaB, 1.0);

    gl_FragColor = resultMask;
}
```
从代码的行数可以看出，这个效果应该是里面最复杂的。为了实现残影，我们先让图片随时间做圆周运动。

`vec4 getMask(float time, vec2 textureCoords, float padding)` 这个函数可以计算出，在某个时刻图片的具体位置。通过它我们可以每经过一段时间，去生成一个新的层。

`float maskAlphaProgress(float currentTime, float hideTime, float startTime)` 这个函数可以计算出，某个时刻创建的层，在当前时刻的透明度。

maxAlphaR 、 maxAlphaG 、 maxAlphaB 分别指定了新层初始的三个颜色通道的透明度。因为最终的效果是残留红色，所以主要保留了红色通道的值。

然后是叠加，和两层叠加的情况类似，这里通过 for 循环来累加每一层的每个通道乘上自身的透明度的值，算出最终的颜色值 resultMask 。

> 注： 在 iOS 的模拟器上，只能用 CPU 来模拟 GPU 的功能。所以在模拟器上运行上面的代码时，可能会十分卡顿。尤其是最后这个效果，由于计算量太大，亲测模拟器显示不出来。因此如果要跑代码，最好使用真机运行。

## 源码

请到 GitHub (https://github.com/lmf12/blog-demo/tree/master/testOpenGLESFilter) 上查看完整代码。

### 参考

1. https://www.jianshu.com/p/5bb7f2a0da90
2. http://www.lymanli.com/2019/02/17/ios-opengles-render-texture/
3. https://baike.baidu.com/item/%E6%B7%B7%E5%90%88%E6%A8%A1%E5%BC%8F/6700481
4. https://gist.github.com/patriciogonzalezvivo/670c22f3966e662d2f83
5. https://xiaoiver.github.io/coding/2018/08/01/%E5%99%AA%E5%A3%B0%E7%9A%84%E8%89%BA%E6%9C%AF.html


