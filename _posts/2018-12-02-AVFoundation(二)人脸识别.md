---
layout: post
title: 'AVFoundation(二)人脸识别'
subtitle: '人脸识别'
date: 2018-12-02
categories: 技术
cover: 
tags: AVFoundation
---

### 视频录制实现

* 关于QuckTime
    * 视频内容的捕捉：当设置捕捉会话时，添加一个名为AVCaptureMovieFileOutput的输出。这个类定义了方法将QuckTime影片捕捉到磁盘。这个类大多数核心功能继承于父类AVCaptureFileOutput。这个父类定义了许多实用功能，比如录制到最长时限或录制到特定文件大小时为止。还可以配置成保留最小可用的磁盘空间，这一点在存储空间有限的移动设备上录制视频时非常重要
    * 通常当QuckTime影片准备发布时，影片头的元数据处于文件的开始位置。这样可以让那个视频播放器快速读取头包含信息，来确定文件的内容、结构和其包含的多个样本的位置。不过，当录制一个QuckTime影片时，直到所有的样片都完成捕捉后才能创建信息头。当录制结束时，创建头数据并将它附在文件结尾处。

    ![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20191202-01.jpg)
    
    * 将创建头过程放在所有影片样本完成捕捉之后存在一个问题，尤其是在移动设备的情况下。如果遇到崩溃或者其他中断，比如电话拨入，则影片头就不会被正确写入，会在磁盘生成一个不可读的影片文件。AVCaptureMovieFileOutput提供一个核心功能就是分段捕捉QuickTime影片。

    ![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20191202-02.jpg)

<pre><code class="language-objectivec">#pragma mark - Video Capture Methods 捕捉视频

//判断是否录制状态
- (BOOL)isRecording {
    return self.movieOutput.isRecording;
}

//开始录制
- (void)startRecording {

    if (![self isRecording]) {
        
        //获取当前视频捕捉连接信息，用于捕捉视频数据配置一些核心属性
        AVCaptureConnection * videoConnection = [self.movieOutput connectionWithMediaType:AVMediaTypeVideo];
        
        //判断是否支持设置videoOrientation 属性。
        if([videoConnection isVideoOrientationSupported])
        {
            //支持则修改当前视频的方向
            videoConnection.videoOrientation = [self currentVideoOrientation];
        }
        
        //判断是否支持视频稳定 可以显著提高视频的质量。只会在录制视频文件涉及
        if([videoConnection isVideoStabilizationSupported])
        {
            videoConnection.enablesVideoStabilizationWhenAvailable = YES;
        }
        
        AVCaptureDevice *device = [self activeCamera];
        
        //摄像头可以进行平滑对焦模式操作。即减慢摄像头镜头对焦速度。当用户移动拍摄时摄像头会尝试快速自动对焦。
        if (device.isSmoothAutoFocusEnabled) {
            NSError *error;
            if ([device lockForConfiguration:&error]) {
                
                device.smoothAutoFocusEnabled = YES;
                [device unlockForConfiguration];
            }else
            {
                [self.delegate deviceConfigurationFailedWithError:error];
            }
        }
        
        //查找写入捕捉视频的唯一文件系统URL.
        self.outputURL = [self uniqueURL];
        
        //开始录制。直播/小视频 ---> 捕捉到视频信息 ---> 压缩(AAC/H264)
        //录制成一个QuckTime视频文件 --> 存储到相册。(也涉及到编码--硬编码--由于AVFoundation已经替你完成)
        //在捕捉输出上调用方法 参数1:录制保存路径  参数2:代理
        [self.movieOutput startRecordingToOutputFileURL:self.outputURL recordingDelegate:self];
    }
}

- (CMTime)recordedDuration {
    
    return self.movieOutput.recordedDuration;
}


//写入视频唯一文件系统URL
- (NSURL *)uniqueURL {

    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    //temporaryDirectoryWithTemplateString  可以将文件写入的目的创建一个唯一命名的目录；
    //临时文件，没下载完成前是没有后缀名的，下载完成后才会变成正确的后缀名
    NSString *dirPath = [fileManager temporaryDirectoryWithTemplateString:@"kamera.XXXXXX"];
    
    if (dirPath) {
        //mov.视频封装的容器。和视频编码格式有区别
        NSString *filePath = [dirPath stringByAppendingPathComponent:@"kamera_movie.mov"];
        return  [NSURL fileURLWithPath:filePath];
        
    }
    
    return nil;
    
}

//停止录制
- (void)stopRecording {

    //是否正在录制
    if ([self isRecording]) {
        [self.movieOutput stopRecording];
    }
}
</code></pre>

### 视频拍摄缩略图实现

<pre><code class="language-objectivec">#pragma mark - AVCaptureFileOutputRecordingDelegate 捕捉文件输出

- (void)captureOutput:(AVCaptureFileOutput *)captureOutput didFinishRecordingToOutputFileAtURL:(NSURL *)outputFileURL fromConnections:(NSArray *)connections error:(NSError *)error {

    //错误
    if (error) {
        [self.delegate mediaCaptureFailedWithError:error];
    }else
    {
        //写入
        [self writeVideoToAssetsLibrary:[self.outputURL copy]];
        
    }
    self.outputURL = nil;
}

//写入捕捉到的视频到相册
- (void)writeVideoToAssetsLibrary:(NSURL *)videoURL {
    
    //ALAssetsLibrary 实例 提供写入视频的接口
    ALAssetsLibrary *library = [[ALAssetsLibrary alloc]init];
    
    //写资源库写入前，检查视频是否可被写入 （写入前尽量养成判断的习惯）
    if ([library videoAtPathIsCompatibleWithSavedPhotosAlbum:videoURL]) {
        
        //创建block块
        ALAssetsLibraryWriteVideoCompletionBlock completionBlock;
        completionBlock = ^(NSURL *assetURL,NSError *error)
        {
            if (error) {
                [self.delegate assetLibraryWriteFailedWithError:error];
            }else
            {
                //用于界面展示视频缩略图
                [self generateThumbnailForVideoAtURL:videoURL];
            }
            
        };
        
        //执行实际写入资源库的动作
        [library writeVideoAtPathToSavedPhotosAlbum:videoURL completionBlock:completionBlock];
    }
}

//获取视频左下角缩略图
- (void)generateThumbnailForVideoAtURL:(NSURL *)videoURL {

    //在videoQueue 上，
    dispatch_async(self.videoQueue, ^{
        
        //建立新的AVAsset & AVAssetImageGenerator
        AVAsset *asset = [AVAsset assetWithURL:videoURL];
        
        AVAssetImageGenerator *imageGenerator = [AVAssetImageGenerator assetImageGeneratorWithAsset:asset];
        
        //设置maximumSize 宽为100，高为0 根据视频的宽高比来计算图片的高度
        imageGenerator.maximumSize = CGSizeMake(100.0f, 0.0f);
        
        //捕捉视频缩略图会考虑视频的变化（如视频的方向变化），如果不设置，缩略图的方向可能出错
        imageGenerator.appliesPreferredTrackTransform = YES;
        
        //获取CGImageRef图片 注意需要自己管理它的创建和释放
        CGImageRef imageRef = [imageGenerator copyCGImageAtTime:kCMTimeZero actualTime:NULL error:nil];
        
        //将图片转化为UIImage
        UIImage *image = [UIImage imageWithCGImage:imageRef];
        
        //释放CGImageRef imageRef 防止内存泄漏
        CGImageRelease(imageRef);
        
        //回到主线程
        dispatch_async(dispatch_get_main_queue(), ^{
            
            //发送通知，传递最新的image
            [self postThumbnailNotifification:image];
        });
    });
}
</code></pre>

### iOS上人脸识别的策略分析

* 框架
    * CoreImage  
    * face++ 
    * OpenCV
    * libefacedetection
    * AVFoundation
    * vision  

* 静态人脸识别(识别图片上的人脸)
* 动态人脸识别(识别拍摄过程中的人脸，实现激萌效果)
* 识别二维码

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20191202-03.jpg)

![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20191202-04.jpg)

### AVFoundation人脸识别

<pre><code class="language-objectivec">- (BOOL)setupSessionOutputs:(NSError **)error {
    
    self.metadataOutput = [[AVCaptureMetadataOutput alloc]init];

    //为捕捉会话添加设备
    if ([self.captureSession canAddOutput:self.metadataOutput]){
        [self.captureSession addOutput:self.metadataOutput];
        
        //获得人脸属性
        NSArray *metadatObjectTypes = @[AVMetadataObjectTypeFace];
        //设置metadataObjectTypes 指定对象输出的元数据类型。
        /*
         限制检查到元数据类型集合的做法是一种优化处理方法！！可以减少我们实际感兴趣的对象数量
         支持多种元数据。这里只保留对人脸元数据感兴趣
         */
        self.metadataOutput.metadataObjectTypes = metadatObjectTypes;
        
        //创建主队列： 因为人脸检测用到了硬件加速GPU，而且许多重要的任务都在主线程中执行，所以需要为这次参数指定主队列。
        dispatch_queue_t mainQueue = dispatch_get_main_queue();
        
        //通过设置AVCaptureVideoDataOutput的代理，就能获取捕获到一帧一帧数据
        [self.metadataOutput setMetadataObjectsDelegate:self queue:mainQueue];
     
        return YES;
    }else
    {
        //报错
        if (error) {
            NSDictionary *userInfo = @{NSLocalizedDescriptionKey:@"Failed to still image output"};
            
            *error = [NSError errorWithDomain:THCameraErrorDomain code:THCameraErrorFailedToAddOutput userInfo:userInfo];
        }
        return NO;
    }
}


//代理方法=捕获到你设置的元数据对象
- (void)captureOutput:(AVCaptureOutput *)captureOutput
didOutputMetadataObjects:(NSArray *)metadataObjects
       fromConnection:(AVCaptureConnection *)connection {

    //metadataObjects 包含了捕获到人脸数据.(人脸数据重复，上一秒捕获到的人脸位置，下一秒还是捕获)
    //使用循环，打印人脸数据
    for (AVMetadataFaceObject *face in metadataObjects) {
        //faceID、bounds唯一。哪怕是双胞胎
        NSLog(@"Face detected with ID:%li",(long)face.faceID);
        NSLog(@"Face bounds:%@",NSStringFromCGRect(face.bounds));
    }
    //已经获取视频中的人脸个数！人脸的e位置！处理人脸！预览图层上
    //将元数据 传递给 THPreviewView.m   将元数据转换为layer
    [self.faceDetectionDelegate didDetectFaces:metadataObjects];
}

/*
 1.视频采集
 2.为session添加一个元数据的输出.ACCaptureMetadataOutput
 3.设置元数据的范围(人脸数据、二维码数据、一维码...)
 4.开始捕捉(设置捕捉代理)didOutPutMetadataObjects
 5.获取到捕捉人脸相关信息：代理方法中可以获取 didOutputMetadataObjects
 6.对人脸数据的处理！将人脸框出来！(涉及比较多的细节)
*/
</code></pre>


### [Yaw,Pitch,Roll](http://blog.sina.com.cn/s/blog_8e6c5b220102wtpd.html)

* 欧拉角是什么？
    * 欧拉角是由3个角组成，这3个角分别是yaw,Pitch,Roll。很难翻译这3个词。Yaw
表示绕Y轴旋转的角度，Pitch表示绕X轴旋转角度，Roll表示绕Z轴旋转的角度。
    * Yaw 偏移
    * Pitch 投掷、倾斜、坠落
    * Roll 转动

    ![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20191202-05.jpg)
    
* 欧拉角中的旋转
    * 这是绕相关轴旋转，乘以相关矩阵即可

    ![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E6%BD%AD%E5%B7%9E/AVFoundation/20191202-06.jpg)

<pre><code class="language-objectivec">- (void)setupView {

    //初始化faceLayers属性  字典，用来记录人脸图层
    self.faceLayers = [NSMutableDictionary dictionary];
    
    //设置videoGravity 使用AVLayerVideoGravityResizeAspectFill 铺满整个预览层的边界范围
    self.previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
    
    //初始化overlayLayer
    self.overlayLayer = [CALayer layer];
    
    //设置它的frame
    self.overlayLayer.frame = self.bounds;
    //假设你的图层上的图形发生3D变换，设置投影方式
    //子图层形变 sublayerTransform属性   Core  Animation动画
    self.overlayLayer.sublayerTransform = CATransform3DMakePerspective(1000);
    
    //将子图层添加到预览图层来
    [self.previewLayer addSublayer:self.overlayLayer];
    
}

//将检测到的人脸进行可视化
- (void)didDetectFaces:(NSArray *)faces {

    //人脸数据位置信息(摄像头坐标系) --> 屏幕坐标系
    //创建一个本地数组 保存转换后的人脸数据
    NSArray *transformedFaces = [self transformedFacesFromFaces:faces];
    
    //获取faceLayers的key，用于确定哪些人移除了视图并将对应的图层移出界面。
    /*
        支持同时识别10个人脸
     如果这个人脸从摄像头消失了！删除它的图层 faceID
     先假设所有的人脸都需要删除，然后再一一从删除列表中移除，相当于从黑名单里面拉出来
     */
    NSMutableArray *lostFaces = [self.faceLayers.allKeys mutableCopy];
    
    //遍历每个转换的人脸对象
    for (AVMetadataFaceObject *face in transformedFaces) {
        
        //获取关联的faceID。这个属性唯一标识一个检测到的人脸
        NSNumber *faceID = @(face.faceID);
        //facaID存在！人脸没有移y出摄像头，不需要删除
        //将对象从lostFaces 移除
        [lostFaces removeObject:faceID];
        
        //拿到当前faceID对应的layer。 old face
        CALayer *layer = self.faceLayers[faceID];
        
        //如果给定的faceID 没有找到对应的图层。new face
        if (!layer) {
            
            //调用makeFaceLayer 创建一个新的人脸图层
            layer = [self makeFaceLayer];
            
            //将新的人脸图层添加到 overlayLayer上
            [self.overlayLayer addSublayer:layer];
            
            //将layer加入到字典中
            self.faceLayers[faceID] = layer;
            
        }
        /*
         人脸是立体的 3D 属性
         */
        //设置图层的transform属性 CATransform3DIdentity 图层默认变化 这样可以重新设置之前应用的变化
        layer.transform = CATransform3DIdentity;
        
        //根据人脸的bounds 设置layer的frame     图层的大小 = 人脸的大小
        layer.frame = face.bounds;
        
        //判断人脸对象是否具有有效的斜倾交。可以理解为人的头部向肩膀方向j倾斜
        if (face.hasRollAngle) {//Yaw即人脸左右晃动，z轴
            
            //如果为YES,则获取相应的CATransform3D 值
            CATransform3D t = [self transformForRollAngle:face.rollAngle];
            //不能直接覆盖，要连接起来。即矩阵相乘
            //将它与标识变化关联在一起，并设置transform属性
            layer.transform = CATransform3DConcat(layer.transform, t);
        }
        
        
        //判断人脸对象是否具有有效的偏转角
        if (face.hasYawAngle) {//y轴，要考虑设备本身角度
            
            //如果为YES,则获取相应的CATransform3D 值
            CATransform3D  t = [self transformForYawAngle:face.yawAngle];
            layer.transform = CATransform3DConcat(layer.transform, t);
            
        }
    }
    
    //处理那些已经从镜头中消失的人脸图层
    //人脸已经消失，但是它对应的图层并没有从界面上随之删除
    //遍历数组将剩下的人脸ID集合从上一个图层和faceLayers字典中移除
    for (NSNumber *faceID in lostFaces) {
        
        CALayer *layer = self.faceLayers[faceID];
        [layer removeFromSuperlayer];
        [self.faceLayers  removeObjectForKey:faceID];
    }
}


//将设备的坐标空间的人脸转换为视图空间的对象集合
- (NSArray *)transformedFacesFromFaces:(NSArray *)faces {

    NSMutableArray *transformeFaces = [NSMutableArray array];
    
    for (AVMetadataObject *face in faces) {
        
        //将摄像头的人脸数据 转换为 视图上的可展示的数据
        //简单说：UIKit的坐标 与 摄像头坐标系统（0，0）-（1，1）不一样。所以需要转换
        //转换需要考虑图层、镜像、视频重力、方向等因素 在iOS6.0之前需要开发者自己计算，但iOS6.0后提供方法
        AVMetadataObject *transformedFace = [self.previewLayer transformedMetadataObjectForMetadataObject:face];
        
        //转换成功后，加入到数组中
        [transformeFaces addObject:transformedFace];
    }
    return transformeFaces;
}

- (CALayer *)makeFaceLayer {

    //创建一个layer
    CALayer *layer = [CALayer layer];
    
    //边框宽度为5.0f
    layer.borderWidth = 5.0f;
    
    //边框颜色为红色
    layer.borderColor = [UIColor redColor].CGColor;
    layer.contents = (id)[UIImage imageNamed:@"551.png"].CGImage;
    
    //返回layer
    return layer;
}



//将 RollAngle 的 rollAngleInDegrees 值转换为 CATransform3D
- (CATransform3D)transformForRollAngle:(CGFloat)rollAngleInDegrees {

    //将人脸对象得到的RollAngle 单位“度” 转为Core Animation需要的弧度值
    CGFloat rollAngleInRadians = THDegreesToRadians(rollAngleInDegrees);

    //将结果赋给CATransform3DMakeRotation x,y,z轴为0，0，1 得到绕Z轴倾斜角旋转转换
    return CATransform3DMakeRotation(rollAngleInRadians, 0.0f, 0.0f, 1.0f);
    //最终会产生矩阵
}


//将 YawAngle 的 yawAngleInDegrees 值转换为 CATransform3D
- (CATransform3D)transformForYawAngle:(CGFloat)yawAngleInDegrees {

    //将角度转换为弧度值
     CGFloat yawAngleInRaians = THDegreesToRadians(yawAngleInDegrees);
    
    //将结果CATransform3DMakeRotation x,y,z轴为0，-1，0 得到绕Y轴选择。
    //由于overlayer 需要应用sublayerTransform，所以图层会投射到z轴上，人脸从一侧转向另一侧会有3D 效果
    CATransform3D yawTransform = CATransform3DMakeRotation(yawAngleInRaians, 0.0f, -1.0f, 0.0f);
    
    //因为应用程序的界面固定为垂直方向，但需要为设备方向计算一个相应的旋转变换
    //如果不这样，会造成人脸图层的偏转效果不正确
    
    return CATransform3DConcat(yawTransform, [self orientationTransform]);
}

- (CATransform3D)orientationTransform {
    //人脸蠢的效果一般就是没有考虑这个
    CGFloat angle = 0.0;
    //拿到设备方向
    switch ([UIDevice currentDevice].orientation) {
            
            //方向：下
        case UIDeviceOrientationPortraitUpsideDown:
            angle = M_PI;
            break;
            
            //方向：右
        case UIDeviceOrientationLandscapeRight:
            angle = -M_PI / 2.0f;
            break;
        
            //方向：左
        case UIDeviceOrientationLandscapeLeft:
            angle = M_PI /2.0f;
            break;

            //其他
        default:
            angle = 0.0f;
            break;
    }
    return CATransform3DMakeRotation(angle, 0.0f, 0.0f, 1.0f);
}

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunused"


static CGFloat THDegreesToRadians(CGFloat degrees) {

    return degrees * M_PI / 180;
}


static CATransform3D CATransform3DMakePerspective(CGFloat eyePosition) {
    
    //CATransform3D 图层的旋转，缩放，偏移，歪斜和应用的透
    //CATransform3DIdentity是单位矩阵，该矩阵没有缩放，旋转，歪斜，透视。该矩阵应用到图层上，就是设置默认值。
    CATransform3D  transform = CATransform3DIdentity;
    
    //透视效果（就是近大远小），是通过设置m34 m34 = -1.0/D 默认是0.D越小透视效果越明显
    //D:eyePosition 观察者到投射面的距离
    transform.m34 = -1.0/eyePosition;
    
    return transform;
}
</code></pre>


