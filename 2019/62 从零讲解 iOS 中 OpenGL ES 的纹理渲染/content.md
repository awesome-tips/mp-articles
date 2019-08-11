本文主要介绍，如何使用 OpenGL ES 来渲染一张图片。内容包括：基础概念的讲解，如何使用 GLKit 来渲染纹理，如何使用 GLSL 编写的着色器来渲染纹理。

## 前言

OpenGL（Open Graphics Library）是 Khronos Group （一个图形软硬件行业协会，该协会主要关注图形和多媒体方面的开放标准）开发维护的一个规范，它是硬件无关的。它主要为我们定义了用来操作图形和图片的一系列函数的 API，OpenGL 本身并非 API。

**OpenGL ES（OpenGL for Embedded Systems）**是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。该规范也是由 Khronos Group 开发维护。

OpenGL ES 去除了**四边形（GL_QUADS）**、**多边形（GL_POLYGONS）**等复杂图元，以及许多非绝对必要的特性，剩下最核心有用的部分。可以理解成是一个在移动平台上**能够支持 OpenGL 最基本功能的精简规范**。

目前 iOS 平台支持的有 OpenGL ES 1.0，2.0，3.0。OpenGL ES 3.0 加入了一些新的特性，但是它除了需要 iOS 7.0 以上之外，还需要 iPhone 5S 之后的设备才能支持。出于现有设备的考虑，我们主要使用 OpenGL ES 2.0。

> 注：下文中的 OpenGL ES 均指代 OpenGL ES 2.0。

## 一、概念

### 1、缓存是什么

OpenGL ES 部分运行在 CPU 上，部分运行在 GPU 上，为了协调这两部分的数据交换，定义了缓存（Buffers）的概念。CPU 和 GPU 都有独自控制的内存区域，缓存可以避免数据在这两块内存区域之间进行复制，提高效率。**缓存实际上就是指一块连续的 RAM 。**

### 2、纹理渲染的含义

**纹理**是一个用来保存图像颜色的元素值的缓存，渲染是指将数据生成图像的过程。**纹理渲染**则是将保存在内存中的颜色值等数据，生成图像的过程。

### 3、坐标系

**OpenGL ES 坐标系**

![1](http://)

**OpenGL ES 坐标系**的范围是 -1 ~ 1，是一个三维的坐标系，通常用 X、Y、Z 来表示。Z 轴的正方向指向屏幕外。在不考虑 Z 轴的情况下，左下角为 (-1, -1, 0)，右上角为 (1, 1, 0)。

**纹理坐标系**

![2](http://)

**纹理坐标系**的范围是 0 ~ 1，是一个二维坐标系，横轴称为 S 轴，纵轴称为 T 轴。在坐标系中，点的横坐标一般用 U 表示，点的纵坐标一般用 V 表示。左下角为 (0, 0)，右上角为 (1, 1)。

> 注： UIKit 坐标系的 (0, 0) 点在左上角，其纵轴的方向和纹理坐标系纵轴的方向刚好相反。

### 4、纹理相关的概念

* **纹素（Texel）**：一个图像初始化为一个纹理缓存后，每个像素会变成一个纹素。纹理的坐标是范围是 0 ~ 1，在这个单位长度内，可能包含任意多个纹素。
* **光栅化（Rasterizing）**：将几何形状数据转换为片段的渲染步骤。
* **片段（Fragment）**：视口坐标中的颜色像素。没有使用纹理时，会使用对象顶点来计算片段的颜色；使用纹理时，会根据纹素来计算。
* **映射（Mapping）**：对齐顶点和纹素的方式。即将顶点坐标 (X, Y, Z) 与 纹理坐标 (U, V) 对应起来。
* **取样（Sampling）**：在顶点固定后，每个片段根据计算出来的 (U, V) 坐标，去找相应纹素的过程。
* **帧缓存（Frame Buffer）**：一个接收渲染结果的缓冲区，为 GPU 指定存储渲染结果的区域。更通俗点，可以理解成存储屏幕上最终显示的一帧画面的区域。

> 注：(U, V) 可能会超出 0 ~ 1 这个范围，需要通过 glTextParameteri() 配置相应的方案，来映射到 S 轴和 T 轴。

### 5、怎么使用缓存

在实际应用中，我们需要使用各种各样的缓存。比如在纹理渲染之前，需要生成一块保存了图像数据的纹理缓存。下面介绍一下缓存管理的一般步骤：

使用缓存的过程可以分为 7 步：

1. **生成（Generate）**：生成缓存标识符 glGenBuffers()
2. **绑定（Bind）**：对接下来的操作，绑定一个缓存 glBindBuffer()
3. **缓存数据（Buffer Data）**：从CPU的内存复制数据到缓存的内存 glBufferData() / glBufferSubData()
4. **启用（Enable）或者禁止（Disable）**：设置在接下来的渲染中是否要使用缓存的数据 glEnableVertexAttribArray() / glDisableVertexAttribArray()
5. **设置指针（Set Pointers）**：告知缓存的数据类型，及相应数据的偏移量 glVertexAttribPointer()
6. **绘图（Draw）**：使用缓存的数据进行绘制 glDrawArrays() / glDrawElements()
7. **删除（Delete）**：删除缓存，释放资源 glDeleteBuffers()

这 7 步很重要，现在先有个印象，后面我们在实际例子中会反复用到。

### 6、OpenGL ES 的上下文

OpenGL ES 是一个状态机，相关的配置信息会被保存在一个**上下文（Context）**中，这个些值会被一直保存，直到被修改。但我们可以配置多个上下文，通过调用 [EAGLContext setCurrentContext:context] 来切换。

### 7、OpenGL ES 中的图元

**图元（Primitive）**是指 OpenGL ES 中支持渲染的基本图形。OpenGL ES 只支持三种图元，分别是顶点、线段、三角形。复杂的图形得通过渲染多个三角形来实现。

### 8、怎么渲染三角形

![3](http://)

渲染三角形的基本流程按照上图所示。其中，**顶点着色器**和**片段着色器**是可编程的部分，着色器（Shader）是一个小程序，它们运行在 GPU 上，在主程序运行的时候进行动态编译，而不用写死在代码里面。编写着色器用的语言是 GLSL（OpenGL Shading Language），在第三节中我们会详细介绍。

下面介绍一下渲染流程的每一步都做了什么：

**顶点数据**

为了渲染一个三角形，我们需要传入一个包含 3 个三维顶点坐标的数组，每个顶点都有对应的顶点属性，顶点属性中可以包含任何我们想用的数据。在上图的例子里，我们的每个顶点包含了一个颜色值。

并且，为了让 OpenGL ES 知道我们是要绘制三角形，而不是点或者线段，我们在调用绘制指令的时候，都会把图元信息传递给 OpenGL ES 。

**顶点着色器**

顶点着色器会对每个顶点执行一次运算，它可以使用顶点数据来计算该顶点的坐标、颜色、光照、纹理坐标等。

顶点着色器的一个重要任务是进行坐标转换，例如将模型的原始坐标系（一般是指其 3D 建模工具中的坐标）转换到屏幕坐标系。

**图元装配**

在顶点着色器程序输出顶点坐标之后，各个顶点按照绘制命令中的图元类型参数，以及顶点索引数组被组装成一个个图元。

通过这一步，模型中 3D 的图元已经被转化为屏幕上 2D 的图元。

**几何着色器**

在「OpenGL」的版本中，顶点着色器和片段着色器之间有一个可选的着色器，叫做几何着色器（Geometry Shader）。

几何着色器把图元形式的一系列顶点的集合作为输入，它可以通过产生新顶点构造出新的图元来生成其他形状。

OpenGL ES 目前还不支持几何着色器，这个部分我们可以先不关注。

**光栅化**

在光栅化阶段，基本图元被转换为供片段着色器使用的片段。片段表示可以被渲染到屏幕上的像素，它包含位置、颜色、纹理坐标等信息，这些值是由图元的顶点信息进行插值计算得到的。

在片段着色器运行之前会执行裁切，处于视图以外的所有像素会被裁切掉，用来提升执行效率。

**片段着色器**

片段着色器的主要作用是计算每一个片段最终的颜色值（或者丢弃该片段）。片段着色器决定了最终屏幕上每一个像素点的颜色值。

**测试与混合**

在这一步，OpenGL ES 会根据片段是否被遮挡、视图上是否已存在绘制好的片段等情况，对片段进行丢弃或着混合，最终被保留下来的片段会被写入帧缓存中，最终呈现在设备屏幕上。

### 9、怎么渲染多变形

由于 OpenGL ES 只能渲染三角形，因此多边形需要由多个三角形来组成。

![4](http://)

如图所示，一个五边形，我们可以把它拆分成 3 个三角形来渲染。

渲染一个三角形，我们需要一个保存 3 个顶点的数组。这意味着我们渲染一个五边形，需要用 9 个顶点。而且我们可以看到，其中 V0 、 V2 、V3 都是重复的顶点，显得有点冗余。

那么有没有更简单的方式，可以让我们复用之前的顶点呢？答案是肯定的。

在 OpenGL ES 中，对于三角形有 3 种绘制模式。在给定的顶点数组相同的情况下，可以指定我们想要的连接方式。如下图所示：

![5](http://)

**GL_TRIANGLES**

GL_TRIANGLES 就是我们一开始说的方式，没有复用顶点，以每三个顶点绘制一个三角形。第一个三角形使用 V0 、 V1 、V2 ，第二个使用 V3 、 V4 、V5 ，以此类推。如果顶点的个数不是 3 的倍数，那么最后的 1 个或者 2 个顶点会被舍弃。

**GL_TRIANGLE_STRIP**

GL_TRIANGLE_STRIP 在绘制三角形的时候，会复用前两个顶点。第一个三角形依然使用 V0 、 V1 、V2 ，第二个则会使用 V1 、 V2 、V3，以此类推。第 n 个会使用 V(n-1) 、 V(n) 、V(n+1) 。

**GL_TRIANGLE_FAN**

GL_TRIANGLE_FAN 在绘制三角形的时候，会复用第一个顶点和前一个顶点。第一个三角形依然使用 V0 、 V1 、V2 ，第二个则会使用 V0 、 V2 、V3，以此类推。第 n 个会使用 V0 、 V(n) 、V(n+1) 。这种方式看上去像是在绕着 V0 画扇形。

## 二、通过 GLKit 渲染

恭喜你终于看完了枯燥的概念讲解。从这里开始，我们开始会进入实际的例子，用代码来讲解渲染的过程。

在 GLKit 中，苹果爸爸对 OpenGL ES 中的一些操作进行了封装，因此我们使用 GLKit 来渲染会省去一些步骤。

那么好奇的你肯定会问，在「纹理渲染」这件事情上，GLKit 帮我们做了什么呢？

**先不着急，等我们讲完第三节中使用 GLSL 渲染的方式，再来回答这个问题。**

现在，让我们怀着忐忑又期待的心情，来看看 GLKit 是怎么渲染纹理的。

### 1、获取顶点数据

定义顶点数据，用一个三维向量来保存 (X, Y, Z) 坐标，用一个二维向量来保存 (U, V) 坐标：

```c
typedef struct {
    GLKVector3 positionCoord; // (X, Y, Z)
    GLKVector2 textureCoord; // (U, V)
} SenceVertex;
```

初始化顶点数据：

```objc
self.vertices = malloc(sizeof(SenceVertex) * 4); // 4 个顶点
    
self.vertices[0] = (SenceVertex){{-1, 1, 0}, {0, 1}}; // 左上角
self.vertices[1] = (SenceVertex){{-1, -1, 0}, {0, 0}}; // 左下角
self.vertices[2] = (SenceVertex){{1, 1, 0}, {1, 1}}; // 右上角
self.vertices[3] = (SenceVertex){{1, -1, 0}, {1, 0}}; // 右下角
```

退出的时候，记得手动释放内存：

```objc
- (void)dealloc {
    // other code ...
    
    if (_vertices) {
        free(_vertices);
        _vertices = nil;
    }
}
```

### 2、初始化 GLKView 并设置上下文

```objc
// 创建上下文，使用 2.0 版本
EAGLContext *context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    
// 初始化 GLKView
CGRect frame = CGRectMake(0, 100, self.view.frame.size.width, self.view.frame.size.width);
self.glkView = [[GLKView alloc] initWithFrame:frame context:context];
self.glkView.backgroundColor = [UIColor clearColor];
self.glkView.delegate = self;
    
[self.view addSubview:self.glkView];
    
// 设置 glkView 的上下文为当前上下文
[EAGLContext setCurrentContext:self.glkView.context];
```

### 3、加载纹理

使用 GLKTextureLoader 来加载纹理，并用 GLKBaseEffect 保存纹理的 ID ，为后面渲染做准备。

```objc
NSString *imagePath = [[[NSBundle mainBundle] resourcePath] stringByAppendingPathComponent:@"sample.jpg"];
UIImage *image = [UIImage imageWithContentsOfFile:imagePath]; 

NSDictionary *options = @{GLKTextureLoaderOriginBottomLeft : @(YES)};
GLKTextureInfo *textureInfo = [GLKTextureLoader textureWithCGImage:[image CGImage]
                                                           options:options
                                                             error:NULL];
self.baseEffect = [[GLKBaseEffect alloc] init];
self.baseEffect.texture2d0.name = textureInfo.name;
self.baseEffect.texture2d0.target = textureInfo.target;
```

因为纹理坐标系和 UIKit 坐标系的纵轴方向是相反的，所以将 GLKTextureLoaderOriginBottomLeft 设置为 YES，用来消除两个坐标系之间的差异。

> 注：这里如果用 imageNamed: 来读取图片，在反复加载相同纹理的时候，会出现上下颠倒的错误。

### 4、实现 GLKView 的代理方法

在 glkView:drawInRect: 代理方法中，我们要去实现顶点数据和纹理数据的绘制逻辑。这一步是重点，注意观察「缓存管理的 7 个步骤」的具体用法。

代码如下：

```objc
- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
    [self.baseEffect prepareToDraw];
    
    // 创建顶点缓存
    GLuint vertexBuffer;
    glGenBuffers(1, &vertexBuffer);  // 步骤一：生成
    glBindBuffer(GL_ARRAY_BUFFER, vertexBuffer);  // 步骤二：绑定
    GLsizeiptr bufferSizeBytes = sizeof(SenceVertex) * 4;
    glBufferData(GL_ARRAY_BUFFER, bufferSizeBytes, self.vertices, GL_STATIC_DRAW);  // 步骤三：缓存数据
    
    // 设置顶点数据
    glEnableVertexAttribArray(GLKVertexAttribPosition);  // 步骤四：启用或禁用
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, sizeof(SenceVertex), NULL + offsetof(SenceVertex, positionCoord));  // 步骤五：设置指针
    
    // 设置纹理数据
    glEnableVertexAttribArray(GLKVertexAttribTexCoord0);  // 步骤四：启用或禁用
    glVertexAttribPointer(GLKVertexAttribTexCoord0, 2, GL_FLOAT, GL_FALSE, sizeof(SenceVertex), NULL + offsetof(SenceVertex, textureCoord));  // 步骤五：设置指针
    
    // 开始绘制
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);  // 步骤六：绘图
    
    // 删除顶点缓存
    glDeleteBuffers(1, &vertexBuffer);  // 步骤七：删除
    vertexBuffer = 0;
}
```

### 5、开始绘制

我们调用 GLKView 的 display 方法，即可以触发 glkView:drawInRect: 回调，开始渲染的逻辑。

代码如下：

```objc
[self.glkView display];
```

至此，使用 GLKit 实现纹理渲染的过程就介绍完毕了。

是不是觉得意犹未尽，那就赶快进入下一节，了解如何直接通过 GLSL 编写的着色器来渲染纹理。

## 三、通过 GLSL 渲染

在这一小节，我们会讲解在不使用 GLKit 的情况下，怎么实现纹理渲染。我们会着重介绍与 GLKit 渲染不同的部分。

> 注：大家实际去查看 demo 的时候，会发现还是有引入 <GLKit/GLKit.h> 这个头文件。这里主要是为了使用 GLKVector3 、 GLKVector2 这两个类型，当然不使用也是完全可以的。目的是为了和 GLKit 的例子保持数据格式的一致，方便大家把注意力放在两者真正差异的部分。

### 1、着色器编写

首先，我们需要自己编写着色器，包括顶点着色器和片段着色器，使用的语言是 GLSL 。这里对于 GLSL 就不展开讲了，只解释一下我们等下会用到的部分，更详细的语法内容，可以参见 这里。

新建一个文件，一般顶点着色器用后缀 .vsh ，片段着色器用后缀 .fsh （当然你不喜欢这么命名也可以，但是为了方便其他人阅读，最好是还是按照这个规范来），然后就可以写代码了。

顶点着色器的代码如下：

```c
attribute vec4 Position;
attribute vec2 TextureCoords;
varying vec2 TextureCoordsVarying;

void main (void) {
    gl_Position = Position;
    TextureCoordsVarying = TextureCoords;
}
```

片段着色器的代码如下：

```c
precision mediump float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

void main (void) {
    vec4 mask = texture2D(Texture, TextureCoordsVarying);
    gl_FragColor = vec4(mask.rgb, 1.0);
}
```

GLSL 是类 C 语言写成，如果学习过 C 语言，上手是很快的。下面对这两个着色器的代码做一下简单的解释。

attribute 修饰符只存在于顶点着色器中，用于储存每个顶点信息的输入，比如这里定义了 Position 和 TextureCoords ，用于接收顶点的位置和纹理信息。

vec4 和 vec2 是数据类型，分别指四维向量和二维向量。

varying 修饰符指顶点着色器的输出，同时也是片段着色器的输入，要求顶点着色器和片段着色器中都同时声明，并完全一致，则在片段着色器中可以获取到顶点着色器中的数据。

gl_Position 和 gl_FragColor 是内置变量，对这两个变量赋值，可以理解为向屏幕输出片段的位置信息和颜色信息。

precision 可以为数据类型指定默认精度，precision mediump float 这一句的意思是将 float 类型的默认精度设置为 mediump 。

uniform 用来保存传递进来的只读值，该值在顶点着色器和片段着色器中都不会被修改。顶点着色器和片段着色器共享了 uniform 变量的命名空间，uniform 变量在全局区声明，同个 uniform 变量在顶点着色器和片段着色器中都能访问到。

sampler2D 是纹理句柄类型，保存传递进来的纹理。

texture2D() 方法可以根据纹理坐标，获取对应的颜色信息。

那么这两段代码的含义就很明确了，顶点着色器将输入的顶点坐标信息直接输出，并将纹理坐标信息传递给片段着色器；片段着色器根据纹理坐标，获取到每个片段的颜色信息，输出到屏幕。

### 2、纹理的加载

少了 GLKTextureLoader 的相助，我们就只能自己去生成纹理了。生成纹理的步骤比较固定，以下封装成一个方法：

```
- (GLuint)createTextureWithImage:(UIImage *)image {
    // 将 UIImage 转换为 CGImageRef
    CGImageRef cgImageRef = [image CGImage];
    GLuint width = (GLuint)CGImageGetWidth(cgImageRef);
    GLuint height = (GLuint)CGImageGetHeight(cgImageRef);
    CGRect rect = CGRectMake(0, 0, width, height);
    
    // 绘制图片
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    void *imageData = malloc(width * height * 4);
    CGContextRef context = CGBitmapContextCreate(imageData, width, height, 8, width * 4, colorSpace, kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big);
    CGContextTranslateCTM(context, 0, height);
    CGContextScaleCTM(context, 1.0f, -1.0f);
    CGColorSpaceRelease(colorSpace);
    CGContextClearRect(context, rect);
    CGContextDrawImage(context, rect, cgImageRef);

    // 生成纹理
    GLuint textureID;
    glGenTextures(1, &textureID);
    glBindTexture(GL_TEXTURE_2D, textureID);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, imageData); // 将图片数据写入纹理缓存
    
    // 设置如何把纹素映射成像素
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    
    // 解绑
    glBindTexture(GL_TEXTURE_2D, 0);
    
    // 释放内存
    CGContextRelease(context);
    free(imageData);
    
    return textureID;
}
```

### 3、着色器的编译链接

对于写好的着色器，需要我们在程序运行的时候，动态地去编译链接。编译一个着色器的代码也比较固定，这里通过后缀名来区分着色器类型，直接看代码：

```objc
- (GLuint)compileShaderWithName:(NSString *)name type:(GLenum)shaderType {
    // 查找 shader 文件
    NSString *shaderPath = [[NSBundle mainBundle] pathForResource:name ofType:shaderType == GL_VERTEX_SHADER ? @"vsh" : @"fsh"]; // 根据不同的类型确定后缀名
    NSError *error;
    NSString *shaderString = [NSString stringWithContentsOfFile:shaderPath encoding:NSUTF8StringEncoding error:&error];
    if (!shaderString) {
        NSAssert(NO, @"读取shader失败");
        exit(1);
    }
    
    // 创建一个 shader 对象
    GLuint shader = glCreateShader(shaderType);
    
    // 获取 shader 的内容
    const char *shaderStringUTF8 = [shaderString UTF8String];
    int shaderStringLength = (int)[shaderString length];
    glShaderSource(shader, 1, &shaderStringUTF8, &shaderStringLength);
    
    // 编译shader
    glCompileShader(shader);
    
    // 查询 shader 是否编译成功
    GLint compileSuccess;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &compileSuccess);
    if (compileSuccess == GL_FALSE) {
        GLchar messages[256];
        glGetShaderInfoLog(shader, sizeof(messages), 0, &messages[0]);
        NSString *messageString = [NSString stringWithUTF8String:messages];
        NSAssert(NO, @"shader编译失败：%@", messageString);
        exit(1);
    }
    
    return shader;
}
```

顶点着色器和片段着色器同样都需要经过这个编译的过程，编译完成后，还需要生成一个着色器程序，将这两个着色器链接起来，代码如下：

```objc
- (GLuint)programWithShaderName:(NSString *)shaderName {
    // 编译两个着色器
    GLuint vertexShader = [self compileShaderWithName:shaderName type:GL_VERTEX_SHADER];
    GLuint fragmentShader = [self compileShaderWithName:shaderName type:GL_FRAGMENT_SHADER];
    
    // 挂载 shader 到 program 上
    GLuint program = glCreateProgram();
    glAttachShader(program, vertexShader);
    glAttachShader(program, fragmentShader);
    
    // 链接 program
    glLinkProgram(program);
    
    // 检查链接是否成功
    GLint linkSuccess;
    glGetProgramiv(program, GL_LINK_STATUS, &linkSuccess);
    if (linkSuccess == GL_FALSE) {
        GLchar messages[256];
        glGetProgramInfoLog(program, sizeof(messages), 0, &messages[0]);
        NSString *messageString = [NSString stringWithUTF8String:messages];
        NSAssert(NO, @"program链接失败：%@", messageString);
        exit(1);
    }
    return program;
}
```

这样，我们只要将两个着色器命名统一，按照规范添加后缀名。然后将着色器名称传入这个方法，就可以获得一个编译链接好的着色器程序。

有了着色器程序后，我们就需要往程序中传入数据，首先要获取着色器中定义的变量，具体操作如下：

> 注：不同类型的变量获取方式不同。

```objc
GLuint positionSlot = glGetAttribLocation(program, "Position");
GLuint textureSlot = glGetUniformLocation(program, "Texture");
GLuint textureCoordsSlot = glGetAttribLocation(program, "TextureCoords");
```

传入生成的纹理 ID：

```c
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, textureID);
glUniform1i(textureSlot, 0);
```

glUniform1i(textureSlot, 0) 的意思是，将 textureSlot 赋值为 0，而 0 与 GL_TEXTURE0 对应，这里如果写 1，glActiveTexture 也要传入 GL_TEXTURE1 才能对应起来。

设置顶点数据：

```objc
glEnableVertexAttribArray(positionSlot);
glVertexAttribPointer(positionSlot, 3, GL_FLOAT, GL_FALSE, sizeof(SenceVertex), NULL + offsetof(SenceVertex, positionCoord));
```

设置纹理数据：

```objc
glEnableVertexAttribArray(textureCoordsSlot);
glVertexAttribPointer(textureCoordsSlot, 2, GL_FLOAT, GL_FALSE, sizeof(SenceVertex), NULL + offsetof(SenceVertex, textureCoord));
```

### 4、Viewport 的设置
在渲染纹理的时候，我们需要指定 Viewport 的尺寸，可以理解为渲染的窗口大小。调用 glViewport 方法来设置：

```objc
glViewport(0, 0, self.drawableWidth, self.drawableHeight);
```

其中

```objc
// 获取渲染缓存宽度
- (GLint)drawableWidth {
    GLint backingWidth;
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_WIDTH, &backingWidth);
    
    return backingWidth;
}

// 获取渲染缓存高度
- (GLint)drawableHeight {
    GLint backingHeight;
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_HEIGHT, &backingHeight);
    
    return backingHeight;
}
```

### 5、渲染层的绑定

通过以上步骤，我们已经拥有了纹理，以及顶点的位置信息。现在到了最后一步，我们要怎么将缓存与视图关联起来？换句话说，假如屏幕上有两个视图，OpenGL ES 要怎么知道将图像渲染到哪个视图上？

所以我们要进行渲染层绑定。通过 renderbufferStorage:fromDrawable: 来实现：

```objc
- (void)bindRenderLayer:(CALayer <EAGLDrawable> *)layer {
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
}
```

以上代码生成了一个帧缓存和一个渲染缓存，并将渲染缓存挂载到帧缓存上，然后设置渲染缓存的输出层为 layer。

最后，将绑定的渲染缓存呈现到屏幕上：

```objc
[self.context presentRenderbuffer:GL_RENDERBUFFER];
```

至此，使用 GLSL 渲染纹理的关键步骤就结束了。

最终效果：

![6](http://)

综上所述，我们可以回答第二节的问题了，GLKit 主要帮我们做了以下几个点：

* **着色器的编写**：GLKit 内置了简单的着色器，不用我们自己去编写。
* **纹理的加载**：GLKTextureLoader 封装了一个将 Image 转化为 Texture 的方法。
* **着色器的编译链接**：GLKBaseEffect 内部实现了着色器的编译链接过程，我们在使用过程中基本可以忽略「着色器」这个概念。
* **Viewport 的设置**：在渲染纹理的时候，需要指定 Viewport 的大小，GLKView 在调用 display 方法的时候，会在内部去设置。
* **渲染层的绑定**：GLKView 内部会调用 renderbufferStorage:fromDrawable: 将自身的 layer 设置为渲染缓存的输出层。因此，在调用 display 方法的时候，内部会调用 presentRenderbuffer: 去将渲染缓存呈现到屏幕上。

## 源码

请到 GitHub (https://github.com/lmf12/blog-demo/tree/master/testOpenGLESRender) 上查看完整代码。

### 参考

* 《OpenGL ES 应用开发实践指南》 
	https://book.douban.com/subject/24849591/
* 你好，三角形 - LearnOpenGL CN
	https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/
* OpenGL ES iOS 入门篇2 - 绘制一个多边形
	http://www.cnblogs.com/CoderAlex/p/6604605.html
* iOS - OpenGLES 之图片纹理
	https://blog.csdn.net/icetime17/article/details/50993655

