![1](http://)

本文介绍了如何使用 OpenGL ES 来实现长腿功能。学习这个例子可以加深我们对纹理渲染流程的理解。另外，还会着重介绍一下「渲染到纹理」这个新知识点。

> 警告：本文属于进阶教程，阅读前请确保已经熟悉 OpenGL ES 纹理渲染的相关概念，否则强行阅读可能导致走火入魔。传送门
> 
> 注：下文中的 OpenGL ES 均指代 OpenGL ES 2.0。

## 一、效果展示

首先来看一下最终的效果，这个功能简单来说，就是实现了图片的局部拉伸，从逻辑上来说并不复杂。

![2](http://)

## 二、思路

### 1、怎么实现拉伸

我们来回忆一下，我们要渲染一张图片，需要将图片拆分成两个三角形，如下所示：

![3](http://)

如果我们想对图片进行拉伸，很简单，只需要修改一下 4 个顶点坐标的 Y 值即可。

![4](http://)

那么，如果我们只想对图片中间的部分进行拉伸，应该怎么做呢？

其实答案也很容易想到，我们只需要修改一下图片的拆分方式。如下所示，我们把图片拆分成了 6 个三角形，也可以说是 3 个小矩形。这样，我们只需要对中间的小矩形做拉伸处理就可以了。

![5](http://)

### 2、怎么实现重复调整

我们观察上面的动态效果图，可以看到第二次的压缩操作，是基于第一次的拉伸操作的结果来进行的。因此，在每一步我们都需要拿到上一步的结果，作为原始图，进行再次调整。

这里的「原始图」就是一个纹理。换句话说，我们需要将每一次的调整结果，都重新生成一个纹理，供下次调整的时候使用。

这一步是本文的重点，我们会通过「渲染到纹理」的方式来实现，具体的步骤我们在后面会详细介绍。

## 三、为什么要使用 OpenGL ES

可能有人会说：你这个功能平平无奇，就算不懂 OpenGL ES，我用其它方式也能实现呀。

确实，在 iOS 中，我们绘图一般是使用 CoreGraphics。假设我们使用 CoreGraphics，也按照上面的实现思路，对原图进行拆分绘制，重复调整的时候进行重新拼接，目测也是能实现相同的功能。

但是，由于 CoreGraphics 绘图依赖于 CPU，当我们在调节拉伸区域的时候，需要不断地进行重绘，此时 CPU 的占用必然会暴涨，从而引起卡顿。而使用 OpenGL ES 则不存在这样的问题。

## 四、实现拉伸逻辑

从上面我们知道，渲染图片我们需要 8 个顶点，而拉伸逻辑的关键就是顶点坐标的计算，在拿到计算结果后再重新渲染。

计算顶点的关键步骤如下：

```objc
/**
 根据当前控件的尺寸和纹理的尺寸，计算初始纹理坐标

 @param size 原始纹理尺寸
 @param startY 中间区域的开始纵坐标位置 0~1
 @param endY 中间区域的结束纵坐标位置 0~1
 @param newHeight 新的中间区域的高度
 */
- (void)calculateOriginTextureCoordWithTextureSize:(CGSize)size
                                            startY:(CGFloat)startY
                                              endY:(CGFloat)endY
                                         newHeight:(CGFloat)newHeight {
    CGFloat ratio = (size.height / size.width) *
                    (self.bounds.size.width / self.bounds.size.height);
    CGFloat textureWidth = self.currentTextureWidth;
    CGFloat textureHeight = textureWidth * ratio;
    
    // 拉伸量
    CGFloat delta = (newHeight - (endY -  startY)) * textureHeight;
    
    // 判断是否超出最大值
    if (textureHeight + delta >= 1) {
        delta = 1 - textureHeight;
        newHeight = delta / textureHeight + (endY -  startY);
    }
    
    // 纹理的顶点
    GLKVector3 pointLT = {-textureWidth, textureHeight + delta, 0};  // 左上角
    GLKVector3 pointRT = {textureWidth, textureHeight + delta, 0};  // 右上角
    GLKVector3 pointLB = {-textureWidth, -textureHeight - delta, 0};  // 左下角
    GLKVector3 pointRB = {textureWidth, -textureHeight - delta, 0};  // 右下角
    
    // 中间矩形区域的顶点
    CGFloat startYCoord = MIN(-2 * textureHeight * startY + textureHeight, textureHeight);
    CGFloat endYCoord = MAX(-2 * textureHeight * endY + textureHeight, -textureHeight);
    GLKVector3 centerPointLT = {-textureWidth, startYCoord + delta, 0};  // 左上角
    GLKVector3 centerPointRT = {textureWidth, startYCoord + delta, 0};  // 右上角
    GLKVector3 centerPointLB = {-textureWidth, endYCoord - delta, 0};  // 左下角
    GLKVector3 centerPointRB = {textureWidth, endYCoord - delta, 0};  // 右下角
    
    // 纹理的上面两个顶点
    self.vertices[0].positionCoord = pointLT;
    self.vertices[0].textureCoord = GLKVector2Make(0, 1);
    self.vertices[1].positionCoord = pointRT;
    self.vertices[1].textureCoord = GLKVector2Make(1, 1);
    // 中间区域的4个顶点
    self.vertices[2].positionCoord = centerPointLT;
    self.vertices[2].textureCoord = GLKVector2Make(0, 1 - startY);
    self.vertices[3].positionCoord = centerPointRT;
    self.vertices[3].textureCoord = GLKVector2Make(1, 1 - startY);
    self.vertices[4].positionCoord = centerPointLB;
    self.vertices[4].textureCoord = GLKVector2Make(0, 1 - endY);
    self.vertices[5].positionCoord = centerPointRB;
    self.vertices[5].textureCoord = GLKVector2Make(1, 1 - endY);
    // 纹理的下面两个顶点
    self.vertices[6].positionCoord = pointLB;
    self.vertices[6].textureCoord = GLKVector2Make(0, 0);
    self.vertices[7].positionCoord = pointRB;
    self.vertices[7].textureCoord = GLKVector2Make(1, 0);
}
```

## 五、渲染到纹理

上面提到：**我们需要将每一次的调整结果，都重新生成一个纹理，供下次调整的时候使用。**

出于对结果分辨率的考虑，我们不会直接读取当前屏幕渲染结果对应的帧缓存，而是采取「渲染到纹理」的方式，重新生成一个宽度与原图一致的纹理。

这是为什么呢？

假设我们有一张 1000 X 1000 的图片，而屏幕上的控件大小是 100 X 100 ，则纹理渲染到屏幕后，渲染结果对应的渲染缓存的尺寸也是 100 X 100 （暂不考虑屏幕密度）。如果我们这时候直接读取屏幕的渲染结果，最多也只能读到 100 X 100 的分辨率。

这样会导致图片的分辨率下降，所以我们会使用能保持原有分辨率的方式，即「渲染到纹理」。

在这之前，我们都是将纹理直接渲染到屏幕上，关键步骤像这样：

```objc
GLuint renderBuffer; // 渲染缓存
GLuint frameBuffer;  // 帧缓存
    
// 绑定渲染缓存要输出的 layer
glGenRenderbuffers(1, &renderBuffer);
glBindRenderbuffer(GL_RENDERBUFFER, renderBuffer);
[self.context renderbufferStorage:GL_RENDERBUFFER fromDrawable:layer];
    
// 将渲染缓存绑定到帧缓存上
glGenFramebuffers(1, &frameBuffer);
glBindFramebuffer(GL_FRAMEBUFFER, frameBuffer);
glFramebufferRenderbuffer(GL_FRAMEBUFFER,
                          GL_COLOR_ATTACHMENT0,
                          GL_RENDERBUFFER,
                          renderBuffer);
```

我们生成了一个渲染缓存，并把这个渲染缓存挂载到帧缓存的 GL_COLOR_ATTACHMENT0 颜色缓存上，并通过 context 为当前的渲染缓存绑定了输出的 layer 。

其实，如果我们不需要在屏幕上显示我们的渲染结果，也可以直接将数据渲染到另一个纹理上。更有趣的是，这个渲染后的结果，还可以被当成一个普通的纹理来使用。这也是我们实现重复调整功能的基础。

具体操作如下：

```objc
// 生成帧缓存，挂载渲染缓存
GLuint frameBuffer;
GLuint texture;
    
glGenFramebuffers(1, &frameBuffer);
glBindFramebuffer(GL_FRAMEBUFFER, frameBuffer);
    
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, newTextureWidth, newTextureHeight, 0, GL_RGBA, GL_UNSIGNED_BYTE, NULL);
    
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
```

通过对比我们可以发现，这里我们用 Texture 来替换 Renderbuffer ，并且同样是挂载到 GL_COLOR_ATTACHMENT0 上，不过这里就不需要另外再绑定 layer 了。

**另外，我们需要为新的纹理设置一个尺寸，这个尺寸不再受限于屏幕上控件的尺寸，这也是新纹理可以保持原有分辨率的原因。**

这时候，渲染的结果都会被保存在 texture 中，而 texture 也可以被当成普通的纹理来使用。

## 六、保存结果

当我们调整出满意的图片后，需要把它保存下来。这里分为两步，第一步仍然是上面提到的重新生成纹理，第二步就是把纹理转化为图片。

第二步主要通过 glReadPixels 方法来实现，它可以从当前的帧缓存中读取出纹理数据。直接上代码：

```objc
// 返回某个纹理对应的 UIImage，调用前先绑定对应的帧缓存
- (UIImage *)imageFromTextureWithWidth:(int)width height:(int)height {
    int size = width * height * 4;
    GLubyte *buffer = malloc(size);
    glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, buffer);
    CGDataProviderRef provider = CGDataProviderCreateWithData(NULL, buffer, size, NULL);
    int bitsPerComponent = 8;
    int bitsPerPixel = 32;
    int bytesPerRow = 4 * width;
    CGColorSpaceRef colorSpaceRef = CGColorSpaceCreateDeviceRGB();
    CGBitmapInfo bitmapInfo = kCGBitmapByteOrderDefault;
    CGColorRenderingIntent renderingIntent = kCGRenderingIntentDefault;
    CGImageRef imageRef = CGImageCreate(width, height, bitsPerComponent, bitsPerPixel, bytesPerRow, colorSpaceRef, bitmapInfo, provider, NULL, NO, renderingIntent);
    
    // 此时的 imageRef 是上下颠倒的，调用 CG 的方法重新绘制一遍，刚好翻转过来
    UIGraphicsBeginImageContext(CGSizeMake(width, height));
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    free(buffer);
    return image;
}
```

至此，我们已经拿到了 UIImage 对象，可以把它保存到相册里了。

## 源码

请到 GitHub (https://github.com/lmf12/MFSpringView) 上查看完整代码。

### 参考

* iOS 中使用 OpenGL 实现增高功能
	https://www.jianshu.com/p/99ce551bb109
* 学习 OpenGL ES 之渲染到纹理
	https://www.jianshu.com/p/eccb53001670


