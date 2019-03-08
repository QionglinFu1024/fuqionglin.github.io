---
layout: post
title: 'AVFoundation(一)'
subtitle: 'AVFoundation'
date: 2018-12-01
categories: 技术
cover: 
tags: AVFoundation
---

# AVFoundation
* 照片/视频捕捉功能
* 小视频/直播
* 核心类在iOS和macOS上是一致的，macOS上提供了一个截屏功能

### 捕捉会话
* 􏰄􏰅􏰋􏰌捕捉会话: AVCaptureSession
* 捕捉设备􏰄􏰅􏰍􏰎: AVCaptureDevice.如麦克风、摄像头
* 􏰄􏰅􏰍􏰎􏰏􏰐􏰄􏰅􏰍􏰎􏰏􏰐捕捉设备输入: AVCaptureDeviceInput
* 捕捉设备输出：AVCaptureOutput 抽象类
	* AVCaptureStillImageOutput 静态图片
	* AVCaptureMovieFileOutput 视频文件
	* AVCaptureAudioDataOutput 音频数据
	* AVCaptureVideoDataOutput 视频数据
* 捕捉连接：AVCaptureConnection 根据捕捉的输出设备的渲染媒体类型自动建立连接
* 捕捉预览： AVCaptureVideoPreviewLayer

### 视频预览图层

<pre><code class="language-objectivec">//私有方法 用于支持该类定义的不同触摸处理方法。 将屏幕坐标系上的触控点转换为摄像头上的坐标系点
- (CGPoint)captureDevicePointForPoint:(CGPoint)point {
    AVCaptureVideoPreviewLayer *layer =
        (AVCaptureVideoPreviewLayer *)self.layer;
    return [layer captureDevicePointOfInterestForPoint:point];
    /*
     AVCaptureVideoPreviewLayer.定义了两个方法，摄像头/屏幕坐标转换；
     captureDevicePointOfInterestForPoint ：获取屏幕坐标系的CGPoint数据，返回转换得到摄像头设备坐标系的CGPoint数据
     pointForCaptureDevicePointOfInterest : 获取摄像头坐标系的CGPoint数据，返回转换得到的屏幕坐标系CGPoint数据
     */
}
</code></pre>

### 视频捕捉关于AVCaputureSession的配置

* 􏰍􏰴设置**Session**
  * 初始化
  * 设置分辨率
  * 配置输入设备(注意转换为AVCaptureDeviceInput)
  * 配置输入设备包括音频输入、视频输入
  * 配置输出(静态图像输出、视频文件输出)
  * 在为session添加输入输出时，注意一定判断是否能添加。(摄像头并不隶属于任何一个App它是公共设备)

<pre><code class="language-objectivec">- (BOOL)setupSession:(NSError **)error {
    //创建捕捉会话。AVCaptureSession 是捕捉场景的中心枢纽
    self.captureSession = [[AVCaptureSession alloc]init];
    
    /*
     AVCaptureSessionPresetHigh
     AVCaptureSessionPresetMedium
     AVCaptureSessionPresetLow
     AVCaptureSessionPreset640x480
     AVCaptureSessionPreset1280x720
     AVCaptureSessionPresetPhoto
     */
    //设置图像的分辨率
    self.captureSession.sessionPreset = AVCaptureSessionPresetHigh;
    
    /** 添加视频设备  */
    
    //拿到默认视频捕捉设备 iOS系统返回后置摄像头
    AVCaptureDevice *videoDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    
    //将捕捉设备封装成AVCaptureDeviceInput
    //注意：为会话添加捕捉设备，必须将设备封装成AVCaptureDeviceInput对象
    AVCaptureDeviceInput *videoInput = [AVCaptureDeviceInput deviceInputWithDevice:videoDevice error:error];
    
    //判断videoInput是否有效
    if (videoInput)
    {
        //canAddInput：测试是否能被添加到会话中。摄像头是公共设备，不隶属于任何一个app，可能其他app也在使用
        if ([self.captureSession canAddInput:videoInput])
        {
            //将videoInput 添加到 captureSession中
            [self.captureSession addInput:videoInput];
            //记录当前活跃设备
            self.activeVideoInput = videoInput;
        }
    }else
    {
        return NO;
    }
    
    /** 添加音频设备  */
    
    //选择默认音频捕捉设备 即返回一个内置麦克风
    AVCaptureDevice *audioDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
    
    //为这个设备创建一个捕捉设备输入
    AVCaptureDeviceInput *audioInput = [AVCaptureDeviceInput deviceInputWithDevice:audioDevice error:error];
   
    //判断audioInput是否有效
    if (audioInput) {
        
        //canAddInput：测试是否能被添加到会话中
        if ([self.captureSession canAddInput:audioInput])
        {
            //将audioInput 添加到 captureSession中
            [self.captureSession addInput:audioInput];
            //麦克风就一个，无须记录
        }
    }else
    {
        return NO;
    }

    /** 设置输出：照片、视频文件 */
    
    //AVCaptureStillImageOutput 实例 从摄像头捕捉静态图片
    self.imageOutput = [[AVCaptureStillImageOutput alloc]init];
    
    //配置字典：希望捕捉到JPEG格式的图片（存储格式）
    self.imageOutput.outputSettings = @{AVVideoCodecKey:AVVideoCodecJPEG};
    
    //输出连接 判断是否可用，可用则添加到输出连接中去
    if ([self.captureSession canAddOutput:self.imageOutput])
    {
        [self.captureSession addOutput:self.imageOutput];
        
    }
    
    
    //创建一个AVCaptureMovieFileOutput 实例，用于将Quick Time 电影录制到文件系统
    self.movieOutput = [[AVCaptureMovieFileOutput alloc]init];
    
    //输出连接 判断是否可用，可用则添加到输出连接中去
    if ([self.captureSession canAddOutput:self.movieOutput])
    {
        [self.captureSession addOutput:self.movieOutput];
    }
    
    //初始化视频队列
    self.videoQueue = dispatch_queue_create("cc.VideoQueue", NULL);
    
    return YES;
}

- (void)startSession {

    //检查是否处于运行状态,注意取反
    if (![self.captureSession isRunning])
    {
        //使用同步调用会损耗一定的时间，则用异步的方式处理
        dispatch_async(self.videoQueue, ^{
            [self.captureSession startRunning];
        });
        
    }
}

- (void)stopSession {
    
    //检查是否处于运行状态
    if ([self.captureSession isRunning])
    {
        //使用异步方式，停止运行
        dispatch_async(self.videoQueue, ^{
            [self.captureSession stopRunning];
        });
    }

}
</code></pre>



### 实现前后摄像头的改变

<pre><code class="language-objectivec">#pragma mark - Device Configuration   配置摄像头支持的方法
//寻找指定的摄像头设备
- (AVCaptureDevice *)cameraWithPosition:(AVCaptureDevicePosition)position {
    
    //获取可用视频设备
    NSArray *devicess = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];
    
    //遍历可用的视频设备 并返回position 参数值
    for (AVCaptureDevice *device in devicess)
    {
        if (device.position == position) {
            return device;
        }
    }
    return nil;
}

//获取当前活跃设备
- (AVCaptureDevice *)activeCamera {
    //返回当前捕捉会话对应的摄像头的device 属性
    return self.activeVideoInput.device;
}

//返回当前未激活的摄像头
- (AVCaptureDevice *)inactiveCamera {

    //通过查找当前激活摄像头的反向摄像头获得，如果设备只有1个摄像头，则返回nil
       AVCaptureDevice *device = nil;
      if (self.cameraCount > 1)
      {
          if ([self activeCamera].position == AVCaptureDevicePositionBack) {
               device = [self cameraWithPosition:AVCaptureDevicePositionFront];
         }else
         {
             device = [self cameraWithPosition:AVCaptureDevicePositionBack];
         }
     }

    return device;
}

//判断是否有超过1个摄像头可用
- (BOOL)canSwitchCameras {
    return self.cameraCount > 1;
}

//可用视频捕捉设备的数量
- (NSUInteger)cameraCount {
     return [[AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo] count];
}

//切换摄像头
- (BOOL)switchCameras {

    //判断是否有多个摄像头
    if (![self canSwitchCameras])
    {
        return NO;
    }
    
    //获取当前设备的反向设备
    NSError *error;
    AVCaptureDevice *videoDevice = [self inactiveCamera];
    
    //将输入设备封装成AVCaptureDeviceInput
    AVCaptureDeviceInput *videoInput = [AVCaptureDeviceInput deviceInputWithDevice:videoDevice error:&error];
    
    //判断videoInput 是否为nil
    if (videoInput)
    {
        //标注原配置变化开始
        [self.captureSession beginConfiguration];
        
        //将捕捉会话中，原本的捕捉输入设备移除
        [self.captureSession removeInput:self.activeVideoInput];
        //不加上面两句会一直加入失败
        //判断新的设备是否能加入
        if ([self.captureSession canAddInput:videoInput])
        {
            //能加入成功，则将videoInput 作为新的视频捕捉设备
            [self.captureSession addInput:videoInput];
            
            //将获得设备 改为 videoInput
            self.activeVideoInput = videoInput;
        }else
        {
            //如果新设备，无法加入。则将原本的视频捕捉设备重新加入到捕捉会话中
            [self.captureSession addInput:self.activeVideoInput];
        }
        
        //配置完成后，提交 AVCaptureSession commitConfiguration 会分批的将所有变更整合在一起。
        [self.captureSession commitConfiguration];
    }else
    {
        //设备添加错误
        //创建AVCaptureDeviceInput 出现错误，则通知委托来处理该错误
        [self.delegate deviceConfigurationFailedWithError:error];
        return NO;
    }
    
    return YES;
}
</code></pre>

### 实现摄像头自动聚焦功能

* 捕捉设备(聚焦/曝光)

* AVCapture Device 定义了很多方法，让开发者控制ios设备上的摄像头。可以独立调整和锁定摄像头的焦距、曝光、白平衡。对焦和曝光可以基于特定的兴趣点进行设置，使其在应用中实现点击对焦、点击曝光的功能。还可以让你控制设备的LED作为拍照的闪光灯或手电筒的使用

*  每当修改摄像头设备时，一定要先测试修改动作是否能被设备支持。并不是所有的摄像头都支持所有功能，例如前置摄像头就不支持对焦操作，因为它和目标距离一般在一臂之长的距离。但大部分后置摄像头是可以支持全尺寸对焦。尝试应用一个不被支持的动作，会导致异常崩溃。所以修改摄像头设备前，需要判断是否支持

<pre><code class="language-objectivec">#pragma mark - Focus Methods 点击聚焦方法的实现
-(BOOL)cameraSupportsTapToFocus {
    //询问激活中的摄像头是否支持兴趣点对焦
    return [[self activeCamera]isFocusPointOfInterestSupported];
}

-(void)focusAtPoint:(CGPoint)point {
      AVCaptureDevice *device = [self activeCamera];
      //是否支持兴趣点对焦 & 是否自动对焦模式
      if (device.isFocusPointOfInterestSupported && [device isFocusModeSupported:AVCaptureFocusModeAutoFocus]) {
          NSError *error;
          //锁定设备准备配置，如果获得了锁
          //因为配置时，不能让多个对象对他进行更改，所以要对该过程上锁
          if ([device lockForConfiguration:&error]) {
              //将focusPointOfInterest属性设置CGPoint
              device.focusPointOfInterest = point;
              //focusMode 设置为AVCaptureFocusModeAutoFocus
              device.focusMode = AVCaptureFocusModeAutoFocus;
              //释放该锁定
              [device unlockForConfiguration];
          }else{
              //错误时，则返回给错误处理代理
              [self.delegate deviceConfigurationFailedWithError:error];
          }
      }
  }
</code></pre>






### 实现摄像头自动曝光以及锁定曝光

<pre><code class="language-objectivec">#pragma mark - Exposure Methods   点击曝光的方法实现
- (BOOL)cameraSupportsTapToExpose {
    
    //询问设备是否支持对一个兴趣点进行曝光
    return [[self activeCamera] isExposurePointOfInterestSupported];
}

static const NSString *THCameraAdjustingExposureContext;

- (void)exposeAtPoint:(CGPoint)point {

    AVCaptureDevice *device = [self activeCamera];
    
    AVCaptureExposureMode exposureMode =AVCaptureExposureModeContinuousAutoExposure;
    
    //判断是否支持 AVCaptureExposureModeContinuousAutoExposure 模式
    if (device.isExposurePointOfInterestSupported && [device isExposureModeSupported:exposureMode]) {
        
        [device isExposureModeSupported:exposureMode];
        
        NSError *error;
        
        //锁定设备准备配置
        if ([device lockForConfiguration:&error])
        {
            //配置期望值
            device.exposurePointOfInterest = point;
            device.exposureMode = exposureMode;
            
            //判断设备是否支持锁定曝光的模式。
            if ([device isExposureModeSupported:AVCaptureExposureModeLocked]) {
                
                //支持，则使用kvo确定设备的adjustingExposure属性的状态。
                [device addObserver:self forKeyPath:@"adjustingExposure" options:NSKeyValueObservingOptionNew context:&THCameraAdjustingExposureContext];
            }
            
            //释放该锁定
            [device unlockForConfiguration];
            
        }else
        {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {

    //判断context（上下文）是否为THCameraAdjustingExposureContext
    if (context == &THCameraAdjustingExposureContext) {
        
        //获取device
        AVCaptureDevice *device = (AVCaptureDevice *)object;
        
        //判断设备是否不再调整曝光等级，确认设备的exposureMode是否可以设置为AVCaptureExposureModeLocked
        if(!device.isAdjustingExposure && [device isExposureModeSupported:AVCaptureExposureModeLocked])
        {
            //移除作为adjustingExposure 的self，就不会得到后续变更的通知
            [object removeObserver:self forKeyPath:@"adjustingExposure" context:&THCameraAdjustingExposureContext];
            
            //异步方式调回主队列，
            dispatch_async(dispatch_get_main_queue(), ^{
                NSError *error;
                if ([device lockForConfiguration:&error]) {
                    
                    //修改exposureMode
                    device.exposureMode = AVCaptureExposureModeLocked;
                    
                    //释放该锁定
                    [device unlockForConfiguration];
                    
                }else
                {
                    [self.delegate deviceConfigurationFailedWithError:error];
                }
            });
            
        }
        
    }else
    {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
    
    
}

//重新设置对焦&曝光
- (void)resetFocusAndExposureModes {

    AVCaptureDevice *device = [self activeCamera];
    AVCaptureFocusMode focusMode = AVCaptureFocusModeContinuousAutoFocus;
    
    //获取对焦兴趣点 和 连续自动对焦模式 是否被支持
    BOOL canResetFocus = [device isFocusPointOfInterestSupported]&& [device isFocusModeSupported:focusMode];
    
    AVCaptureExposureMode exposureMode = AVCaptureExposureModeContinuousAutoExposure;
    
    //确认曝光度可以被重设
    BOOL canResetExposure = [device isFocusPointOfInterestSupported] && [device isExposureModeSupported:exposureMode];
    
    //回顾一下，捕捉设备空间左上角（0，0），右下角（1，1） 中心点则（0.5，0.5）
    CGPoint centPoint = CGPointMake(0.5f, 0.5f);
    
    NSError *error;
    
    //锁定设备，准备配置
    if ([device lockForConfiguration:&error]) {
        
        //焦点可设，则修改
        if (canResetFocus) {
            device.focusMode = focusMode;
            device.focusPointOfInterest = centPoint;
        }
        
        //曝光度可设，则设置为期望的曝光模式
        if (canResetExposure) {
            device.exposureMode = exposureMode;
            device.exposurePointOfInterest = centPoint;
            
        }
        
        //释放锁定
        [device unlockForConfiguration];
        
    }else
    {
        [self.delegate deviceConfigurationFailedWithError:error];
    }
}
</code></pre>



### 实现摄像头手电筒和闪光灯模式的开启关闭

<pre><code class="language-objectivec">#pragma mark - Flash and Torch Modes    闪光灯 & 手电筒

//判断是否有闪光灯
- (BOOL)cameraHasFlash {
    return [[self activeCamera]hasFlash];
}

//闪光灯模式
- (AVCaptureFlashMode)flashMode {
    return [[self activeCamera]flashMode];
}

//设置闪光灯
- (void)setFlashMode:(AVCaptureFlashMode)flashMode {

    //获取会话
    AVCaptureDevice *device = [self activeCamera];
    
    //判断是否支持闪光灯模式
    if ([device isFlashModeSupported:flashMode]) {
    
        //如果支持，则锁定设备
        NSError *error;
        if ([device lockForConfiguration:&error]) {

            //修改闪光灯模式
            device.flashMode = flashMode;
            //修改完成，解锁释放设备
            [device unlockForConfiguration];
            
        }else
        {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}

//是否支持手电筒
- (BOOL)cameraHasTorch {
    return [[self activeCamera]hasTorch];
}

//手电筒模式
- (AVCaptureTorchMode)torchMode {
    return [[self activeCamera]torchMode];
}

//设置是否打开手电筒
- (void)setTorchMode:(AVCaptureTorchMode)torchMode {

    AVCaptureDevice *device = [self activeCamera];
    
    if ([device isTorchModeSupported:torchMode]) {
        
        NSError *error;
        if ([device lockForConfiguration:&error]) {
            
            device.torchMode = torchMode;
            [device unlockForConfiguration];
        }else
        {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}
</code></pre>



### 静态图片的拍摄

<pre><code class="language-objectivec">#pragma mark - Image Capture Methods 拍摄静态图片
/*
    AVCaptureStillImageOutput 是AVCaptureOutput的子类。用于捕捉图片
 */
- (void)captureStillImage {
    
    //获取连接
    AVCaptureConnection *connection = [self.imageOutput connectionWithMediaType:AVMediaTypeVideo];
    //程序只支持纵向，但是如果用户横向拍照时，需要调整结果照片的方向
    //判断是否支持设置视频方向
    if (connection.isVideoOrientationSupported) {
        
        //获取方向值
        connection.videoOrientation = [self currentVideoOrientation];
    }
    
    //定义一个handler 块，会返回1个图片的NSData数据（完成捕捉之后，我们要做的事情）
    id handler = ^(CMSampleBufferRef sampleBuffer,NSError *error){
                    if (sampleBuffer != NULL) {
                        //从缓冲区，将缓冲数据转换成静态图片
                        NSData *imageData = [AVCaptureStillImageOutput jpegStillImageNSDataRepresentation:sampleBuffer];
                        UIImage *image = [[UIImage alloc]initWithData:imageData];
                        
                        //重点：捕捉图片成功后，将图片传递出去
                        [self writeImageToAssetsLibrary:image];
                    }else
                    {
                        NSLog(@"NULL sampleBuffer:%@",[error localizedDescription]);
                    }
                        
                };
    
    //捕捉静态图片
    [self.imageOutput captureStillImageAsynchronouslyFromConnection:connection completionHandler:handler];
}

//获取方向值
- (AVCaptureVideoOrientation)currentVideoOrientation {
    
    AVCaptureVideoOrientation orientation;
    
    //获取UIDevice 的 orientation
    switch ([UIDevice currentDevice].orientation) {
        case UIDeviceOrientationPortrait:
            orientation = AVCaptureVideoOrientationPortrait;
            break;
        case UIDeviceOrientationLandscapeRight:
            orientation = AVCaptureVideoOrientationLandscapeLeft;
            break;
        case UIDeviceOrientationPortraitUpsideDown:
            orientation = AVCaptureVideoOrientationPortraitUpsideDown;
            break;
        default:
            orientation = AVCaptureVideoOrientationLandscapeRight;
            break;
    }
    
    return orientation;
}

/*
    Assets Library 框架 
    用来让开发者通过代码方式访问iOS photo
    注意：会访问到相册，需要修改plist 权限。否则会导致项目崩溃
 */

- (void)writeImageToAssetsLibrary:(UIImage *)image {

    //创建ALAssetsLibrary  实例
    ALAssetsLibrary *library = [[ALAssetsLibrary alloc]init];
    
    //参数1:图片（参数为CGImageRef 所以image.CGImage）
    //参数2:方向参数 转为NSUInteger
    //参数3:写入成功、失败处理
    [library writeImageToSavedPhotosAlbum:image.CGImage
                             orientation:(NSUInteger)image.imageOrientation
                         completionBlock:^(NSURL *assetURL, NSError *error) {
                             //成功后，发送捕捉图片通知。用于绘制程序的左下角的缩略图
                             if (!error)
                             {
                                 //拍照时，左下角都有一个图片缩略图
                                 [self postThumbnailNotifification:image];
                             }else
                             {
                                 //失败打印错误信息
                                 id message = [error localizedDescription];
                                 NSLog(@"%@",message);
                             }
                         }];
}

//发送缩略图通知
- (void)postThumbnailNotifification:(UIImage *)image {
    
    //回到主队列
    dispatch_async(dispatch_get_main_queue(), ^{
        //发送请求
        NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
        [nc postNotificationName:THThumbnailCreatedNotification object:image];
    });
}
</code></pre>



