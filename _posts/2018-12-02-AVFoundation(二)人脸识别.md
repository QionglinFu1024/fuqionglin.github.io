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


