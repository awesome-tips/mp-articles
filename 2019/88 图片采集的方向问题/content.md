

> 刚开始使用AVFoundation进行采集的时候，经常会发现采集回来的图片方向不对。一般我们都是垂直（HOME键在底部）操作手机，但是在手机用相册或者在电脑上点开采集的图片时，都会发现图片逆时针旋转了90度。为了发现问题的所在，我们需要了解一下通常在图片采集中我们会遇到的各种方向。

## 图片方向 UIImageOrientation

日常开发中，UIImage是一个使用频率很高的类，但是一般我们并不太在意他的方向问题。我们可以通过imageOrientation属性获取UIImage的图片方向，即UIImageOrientation。该方向总共有八种情况，用于表示当前图片的方向状态：

* UIImageOrientationUp: 方向正确
* UIImageOrientationDown: 旋转180度
* UIImageOrientationLeft: 逆时针旋转90度
* UIImageOrientationRight: 顺时针旋转90度
* UIImageOrientationUpMirrored: 水平镜像
* UIImageOrientationDownMirrored: 旋转180度 + 水平镜像
* UIImageOrientationLeftMirrored: 逆时针旋转90度 + 垂直镜像
* UIImageOrientationRightMirrored: 顺时针旋转90度 + 垂直镜像

在平时我们并没有发现UIImage的显示方向有问题，是因为它在显示的时候会根据当前图片的imageOrientation属性，进行相应的转换，即逆操作。如果值是UIImageOrientationLeft则表示图片在显示的时候，需要顺时针旋转90度。

简单来说，UIImageOrientation就是告诉我们该图像的方向属于什么状态，在显示时候是需要做一个逆操作来让图像真正变成我们想要的方向。

回到文章一开始的问题，图片逆时针旋转了90度是因为当前的图像的像素点数据是真的逆时针旋转了90度。虽然整个图片中是有图片方向这个属性的，但是在用文件显示图片的过程中，系统并不会根据该属性进行逆操作，而是耿直的直接显示了。

## 视频方向 AVCaptureVideoOrientation

在进行媒体的采集中，无论是进行拍照和录像，都会有一个相似的操作，即设置连接的videoOrientation属性。系统是不知道你视频图片的正确方向的，只能通过videoOrientation的值来进行图片方向的设置。视频方向一共有四个，这里使用Home键的位置进行解释：

* AVCaptureVideoOrientationPortrait: HOME键在底部
* AVCaptureVideoOrientationPortraitUpsideDown: HOME键在顶部
* AVCaptureVideoOrientationLandscapeRight: HOME键在右边
* AVCaptureVideoOrientationLandscapeLeft: HOME键在左边

> 简单来说，AVCaptureVideoOrientation是不会真正的修改图片像素点的位置，只是给图片设置一个合适的方向（参照当前视频的正确方向）属性。

## 罪魁祸首

在了解了UIImageOrientation和AVCaptureVideoOrientation后，我们就来解答开篇问题，找出引发这个问题的“罪魁祸首”。

下面的讲解中，我简单把图片看成是两个部分组成：

* 图片数据
* 图片方向

在图片采集的时候，图片数据是按照相机获取的数据。平时我们都是都认HOME键在下方是正确的方向，但是对于相机，他正确的方向是摄像头在用户的左上方的情况，即HOME键在右边的情况：

![1](http://)

> PS：图片数据的方向就是相机的方向

因此在videoOrientation为AVCaptureVideoOrientationPortrait的情况下，图片数据效果就是右侧的图片。虽然他的图片方向是UIImageOrientationLeft，但是一般的文件显示都是直接显示他原来的面貌。因此，用户看到的和我们最终保存的图片颠倒“罪魁祸首”就是相机的原始方向与我们的预期不符合。

## 解决方法

图片本身是没错的，错的是显示的时候没有没有按图片方向进行逆操作。那么，我们只要在每次采集到图片的时候对图片进行一次逆操作，让图片数据直接变成与“**图片数据+图片方向**”的效果即可：

```objc
- (UIImage *)fixOrientation
{
    // No-op if the orientation is already correct
    if (self.imageOrientation == UIImageOrientationUp) return self;
    
    // We need to calculate the proper transformation to make the image upright.
    // We do it in 2 steps: Rotate if Left/Right/Down, and then flip if Mirrored.
    CGAffineTransform transform = CGAffineTransformIdentity;
    
    switch (self.imageOrientation) {
        case UIImageOrientationDown:
        case UIImageOrientationDownMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.width, self.size.height);
            transform = CGAffineTransformRotate(transform, M_PI);
            break;
            
        case UIImageOrientationLeft:
        case UIImageOrientationLeftMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.width, 0);
            transform = CGAffineTransformRotate(transform, M_PI_2);
            break;
            
        case UIImageOrientationRight:
        case UIImageOrientationRightMirrored:
            transform = CGAffineTransformTranslate(transform, 0, self.size.height);
            transform = CGAffineTransformRotate(transform, -M_PI_2);
            break;
        case UIImageOrientationUp:
        case UIImageOrientationUpMirrored:
            break;
    }
    
    switch (self.imageOrientation) {
        case UIImageOrientationUpMirrored:
        case UIImageOrientationDownMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.width, 0);
            transform = CGAffineTransformScale(transform, -1, 1);
            break;
            
        case UIImageOrientationLeftMirrored:
        case UIImageOrientationRightMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.height, 0);
            transform = CGAffineTransformScale(transform, -1, 1);
            break;
        case UIImageOrientationUp:
        case UIImageOrientationDown:
        case UIImageOrientationLeft:
        case UIImageOrientationRight:
            break;
    }
    
    // Now we draw the underlying CGImage into a new context, applying the transform
    // calculated above.
    CGContextRef ctx = CGBitmapContextCreate(NULL, self.size.width, self.size.height,
                                             CGImageGetBitsPerComponent(self.CGImage), 0,
                                             CGImageGetColorSpace(self.CGImage),
                                             CGImageGetBitmapInfo(self.CGImage));
    CGContextConcatCTM(ctx, transform);
    switch (self.imageOrientation) {
        case UIImageOrientationLeft:
        case UIImageOrientationLeftMirrored:
        case UIImageOrientationRight:
        case UIImageOrientationRightMirrored:
            // Grr...
            CGContextDrawImage(ctx, CGRectMake(0,0,self.size.height,self.size.width), self.CGImage);
            break;
            
        default:
            CGContextDrawImage(ctx, CGRectMake(0,0,self.size.width,self.size.height), self.CGImage);
            break;
    }
    
    // And now we just create a new UIImage from the drawing context
    CGImageRef cgimg = CGBitmapContextCreateImage(ctx);
    UIImage *img = [UIImage imageWithCGImage:cgimg];
    CGContextRelease(ctx);
    CGImageRelease(cgimg);
    return img;
}
```

> PS：新生成的图片imageOrientation是默认的UIImageOrientationUp

## 采集方向解决方向

有的 App 支持横屏，有的不支持。在方向问题解决上是有两种主要的解决方向的：

* 不支持横屏：
	* 固定死 AVCaptureConnection 的videoOrientation属性，一般就支持AVCaptureVideoOrientationPortrait
	* 在获取图片的时候根据设备当前物理方向([UIDevice currentDevice].orientation)进行变换(CATransform)
* 支持横屏：
	* 在每次确定方向(videoOrientation)的时候，都需要获取最新的videoOrientation

> PS: AVCaptureConnection 在input 发生变化的时候，会跟着变化。比如切换前后摄像头，old input被移除，new input被添加，最后会导致connection的替换。

至此就是我在图片采集遇到的一个问题和解决方法。大家有什么意见或者建议，欢迎在评论区中提出。


