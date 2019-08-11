最近上映的复仇者联盟4据说没有片尾彩蛋，不过谷歌帮我们做了。只要在谷歌搜索灭霸，在结果的右侧点击无限手套，你将化身为灭霸，其中一半的搜索结果会化为灰烬消失...那么这么酷的动画在iOS中可以实现吗？答案是肯定的。整个动画主要包含以下几部分：响指动画、沙化消失以及背景音效和复原动画，让我们分别来看看如何实现。

![1. 沙化动画](http://)

![2. 复原动画](http://)

## 响指动画

Google的方法是利用了48帧合成的一张Sprite图进行动画的：

![3. 响指Sprite图片](http://)

> 原始图片中48幅全部排成一行，这里为了显示效果截成2行

iOS 中通过这张图片来实现动画并不难。CALayer有一个属性contentsRect，通过它可以控制内容显示的区域，而且是Animateable的。它的类型是CGRect，默认值为(x:0.0, y:0.0, width:1.0, height:1.0)，它的单位不是常见的Point，而是单位坐标空间，所以默认值显示100%的内容区域。新建Sprite播放视图层AnimatableSpriteLayer:

```objc
class AnimatableSpriteLayer: CALayer {
    private var animationValues = [CGFloat]()
    convenience init(spriteSheetImage: UIImage, spriteFrameSize: CGSize ) {
        self.init()
        //1
        masksToBounds = true
        contentsGravity = CALayerContentsGravity.left
        contents = spriteSheetImage.cgImage
        bounds.size = spriteFrameSize
        //2
        let frameCount = Int(spriteSheetImage.size.width / spriteFrameSize.width)
        for frameIndex in 0..<frameCount {
            animationValues.append(CGFloat(frameIndex) / CGFloat(frameCount))
        }
    }
    
    func play() {
        let spriteKeyframeAnimation = CAKeyframeAnimation(keyPath: "contentsRect.origin.x")
        spriteKeyframeAnimation.values = animationValues
        spriteKeyframeAnimation.duration = 2.0
        spriteKeyframeAnimation.timingFunction = CAMediaTimingFunction(name: CAMediaTimingFunctionName.linear)
        //3
        spriteKeyframeAnimation.calculationMode = CAAnimationCalculationMode.discrete
        add(spriteKeyframeAnimation, forKey: "spriteKeyframeAnimation")
    }
}
```

* //1: masksToBounds = true和contentsGravity = CALayerContentsGravity.left是为了当前只显示Sprite图的第一幅画面
* //2: 根据Sprite图大小和每幅画面的大小计算出画面数量，预先计算出每幅画面的contentsRect.origin.x偏移量
* //3: 这里是关键，指定关键帧动画的calculationMode为discrete确保关键帧动画依次使用values中指定的关键帧值进行变化，而不是默认情况下采用线性插值进行过渡，来个对比图可能比较容易理解：

![4. 离散模式](http://)

![5. 默认的线性模式](http://)

## 沙化消失

这个效果是整个动画较难的部分，Google的实现很巧妙，它将需要沙化消失内容的html通过html2canvas渲染成canvas，然后将其转换为图片后的每一个像素点随机地分配到32块canvas中，最后对每块画布进行随机地移动和旋转即达到了沙化消失的效果。

### 像素处理

新建自定义视图 DustEffectView，这个视图的作用是用来接收图片并将其进行沙化消失。首先创建函数createDustImages，它将一张图片的像素随机分配到32张等待动画的图片上：

```objc
class DustEffectView: UIView {
    private func createDustImages(image: UIImage) -> [UIImage] {
        var result = [UIImage]()
        guard let inputCGImage = image.cgImage else {
            return result
        }
        //1
        let colorSpace = CGColorSpaceCreateDeviceRGB()
        let width = inputCGImage.width
        let height = inputCGImage.height
        let bytesPerPixel = 4
        let bitsPerComponent = 8
        let bytesPerRow = bytesPerPixel * width
        let bitmapInfo = CGImageAlphaInfo.premultipliedLast.rawValue | CGBitmapInfo.byteOrder32Little.rawValue
        
        guard let context = CGContext(data: nil, width: width, height: height, bitsPerComponent: bitsPerComponent, bytesPerRow: bytesPerRow, space: colorSpace, bitmapInfo: bitmapInfo) else {
            return result
        }
        context.draw(inputCGImage, in: CGRect(x: 0, y: 0, width: width, height: height))
        guard let buffer = context.data else {
            return result
        }
        let pixelBuffer = buffer.bindMemory(to: UInt32.self, capacity: width * height)
        //2
        let imagesCount = 32
        var framePixels = Array(repeating: Array(repeating: UInt32(0), count: width * height), count: imagesCount)
        for column in 0..<width {
            for row in 0..<height {
                let offset = row * width + column
                //3
                for _ in 0...1 { 
                    let factor = Double.random(in: 0..<1) + 2 * (Double(column)/Double(width))
                    let index = Int(floor(Double(imagesCount) * ( factor / 3)))
                    framePixels[index][offset] = pixelBuffer[offset]
                }
            }
        }
        //4
        for frame in framePixels {
            let data = UnsafeMutablePointer(mutating: frame)
            guard let context = CGContext(data: data, width: width, height: height, bitsPerComponent: bitsPerComponent, bytesPerRow: bytesPerRow, space: colorSpace, bitmapInfo: bitmapInfo) else {
                continue
            }
            result.append(UIImage(cgImage: context.makeImage()!, scale: image.scale, orientation: image.imageOrientation))
        }
        return result
    }
}
```

* //1: 根据指定格式创建位图上下文，然后将输入的图片绘制上去之后获取其像素数据
* //2: 创建像素二维数组，遍历输入图片每个像素，将其随机分配到数组32个元素之一的相同位置。随机方法有点特别，原始图片左边的像素只会分配到前几张图片，而原始图片右边的像素只会分配到后几张。

![6. 上部分为原始图片，下部分为像素分配后的32张图片依次显示效果](http://)

* //3: 这里循环2次将像素分配两次，可能 Google 觉得只分配一遍会造成像素比较稀疏。个人认为在移动端，只要一遍就好了。
* //4: 创建32张图片并返回

### 添加动画

Google的实现是给canvas中css的transform属性设置为rotate(deg) translate(px, px) rotate(deg)，值都是随机生成的。如果你对CSS的动画不熟悉，那你会觉得在iOS中只要添加三个CABasicAnimation然后将它们添加到AnimationGroup就好了嘛，实际上并没有那么简单... 因为CSS的transform中后一个变换函数是基于前一个变换后的新transform坐标系。假如某张图片的动画样式是这样的：rotate(90deg) translate(0px, 100px) rotate(-90deg) 直觉告诉我应该是旋转着向下移动100px，然而在CSS中的元素是这么运动的：

![7. CSS中transform多值动画](http://)

第一个rotate和translate决定了最终的位置和运动轨迹，至于第二个rotate作用，只是叠加第一个rotate的值作为最终的旋转弧度，这里刚好为0也就是不旋转。那么在iOS中该如何实现相似的运动轨迹呢？可以利用UIBezierPath, CAKeyframeAnimation的属性path可以指定这个UIBezierPath为动画的运动轨迹。确定起点和实际终点作为贝塞尔曲线的起始点和终止点，那么如何确定控制点？好像可以将“预想”的终点（下图中的(0,-1)）作为控制点。

![8. 将“预想”的终点作为控制点的贝塞尔曲线，看起来和CSS中的运动轨迹差不多](http://)

> 扩展问题
> 通过文章中描述的方式生成的贝塞尔曲线是否与CSS中的动画轨迹完全一致呢？

现在可以给视图添加动画了：

```objc
    let layer = CALayer()
    layer.frame = bounds
    layer.contents = image.cgImage
    self.layer.addSublayer(layer)
    let centerX = Double(layer.position.x)
    let centerY = Double(layer.position.y)
    let radian1 = Double.pi / 12 * Double.random(in: -0.5..<0.5)
    let radian2 = Double.pi / 12 * Double.random(in: -0.5..<0.5)
    let random = Double.pi * 2 * Double.random(in: -0.5..<0.5)
    let transX = 60 * cos(random)
    let transY = 30 * sin(random)
    //1: 
    // x' = x*cos(rad) - y*sin(rad)
    // y' = y*cos(rad) + x*sin(rad)
    let realTransX = transX * cos(radian1) - transY * sin(radian1)
    let realTransY = transY * cos(radian1) + transX * sin(radian1)
    let realEndPoint = CGPoint(x: centerX + realTransX, y: centerY + realTransY)
    let controlPoint = CGPoint(x: centerX + transX, y: centerY + transY)
    //2:
    let movePath = UIBezierPath()
    movePath.move(to: layer.position)
    movePath.addQuadCurve(to: realEndPoint, controlPoint: controlPoint)
    let moveAnimation = CAKeyframeAnimation(keyPath: "position")
    moveAnimation.path = movePath.cgPath
    moveAnimation.calculationMode = .paced
    //3:                 
    let rotateAnimation = CABasicAnimation(keyPath: "transform.rotation")
    rotateAnimation.toValue = radian1 + radian2
    let fadeOutAnimation = CABasicAnimation(keyPath: "opacity")
    fadeOutAnimation.toValue = 0.0
    let animationGroup = CAAnimationGroup()
    animationGroup.animations = [moveAnimation, rotateAnimation, fadeOutAnimation]
    animationGroup.duration = 1
    //4:
    animationGroup.beginTime = CACurrentMediaTime() + 1.35 * Double(i) / Double(imagesCount)
    animationGroup.isRemovedOnCompletion = false
    animationGroup.fillMode = .forwards
    layer.add(animationGroup, forKey: nil)
```

* //1: 实际的偏移量旋转了radian1弧度，这个可以通过公式x' = x*cos(rad) - y*sin(rad), y' = y*cos(rad) + x*sin(rad)算出
* //2: 创建UIBezierPath并关联到CAKeyframeAnimation中
* //3: 两个弧度叠加作为最终的旋转弧度
* //4: 设置CAAnimationGroup的开始时间，让每层Layer的动画延迟开始

## 结尾

到这里，谷歌灭霸彩蛋中较复杂的技术点均已实现。如果您感兴趣，完整的代码(包含音效和复原动画)可以通过文章开头的链接进行查看，可以尝试将沙化图片的数量从32提高至更多，效果会越好，内存也会消耗更多 :-D。

