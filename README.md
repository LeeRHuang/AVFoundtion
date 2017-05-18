目标:
可以正常视频录制并且不掉帧
可以分段视频录制并且不掉帧
可以切换前后摄像头录制视频
可以合成分段录制的视频
视频的导出和存储格式
压缩已经合成的本地视频和上传七牛云存储
entry流播放性能优化
视频播放缓存
测试和预研
* 选填。如果之前做了一些技术调研、竞品分析、多种实现方案对比，请在这里描述。
技术调研:
在前期技术调研过程中，主要参考上述[技术设计目标]来进行，主要涉及以下方向

视频正常录制  iOS视频录制主要使用AVFoundtion框架，有两种实现方案，分别是使用AVCaptureFileOutPutRecordingDelegate和AVCaptureAudioDataOutPutSampleBufferDelegate+AVCaptureVideoDataOutPutSampleBufferDelegate。使用前者是直接从代理回调中获取录制的文件存储地址，开启一个计时器NStimer控制时间和分段录制，比较简单，但是缺点是录制的文件体积比较大，压缩参数设置有限，并且压缩时间比较长; 使用后者稍微麻烦，需要对音频和视频单独处理，实时写入指定的文件路劲中，分段录制时间节点也需要实时计算得出，但是优点很多，在录制时候就可以设置很多音频和视频的压缩参数，可以针对每一帧图片处理，视频体积也比较小，可操作空间更大.
分段录制  由于系统API没有提供暂停的原生支持，需要在每次暂停时候通过获取到的sampleBuffer里面获取到时间信息，记录一个时间戳，再次启动时候重新生成一个文件写入路劲，最后完成时候进行视频合成处理.
切换前后摄像头拍摄  拍摄过程可以暂停，可以切换前后摄像头，需要将视频的方向信息记录下来，在完成时候进行方向的旋转.
视频格式  由于视频需要支持跨平台，所以选择比较主流支持更全面的mp4文件格式
视频压缩  视频压缩，主要考虑的是视频的码率、帧数，音频的采样率.
视频播放 视频播放使用AVFoundtion框架的AVPlayer，原因是UI需要自定义.
视频缓存 播放的视频需要缓存下来，iOS提供一个中间代理AVAssetResourceLoader变下边播.
       
设计详情
* 必填。具体描述设计的内容，自由发挥，文字图标均可。
文字描述:

录制
首先获取相机、麦克风权限
获取前(后)摄像头device
初始化音频、视频输入 deviceInput
初始化中间管理者AVCaptureSession实例和配置
初始化音频、视频输出 deviceOutPut
初始化writer，用于写入获取到的buffer流
初始化预览AVCaptureVideoPreviewLayer
启动session通过前后摄像头捕捉帧画面，实时预览
启动record，通过sampleBuffer代理方法不断捕捉到画面，通过初始化好的writer持续写入到文件中
 暂停清空之前初始化好的writer，再次启动重新生成writer，指定新的文件路劲，记录时间戳，这样就产生了分段视频
录制达到指定时间或者手动到下一步，取出所有录制好的文件，进行拼接压缩，导出至指定格式，完毕

                                                                 

KEPAVCaptureManager
抽象出的拍照和视频基础管理类
KEPCameraExporter
视频数据处理类
KEPCameraUtils
视频工具类



   
播放和缓存
 播放本地视频和流视频都是使用AVPlayer，统一抽象出一个KEPVideoPlayer，在初始化时候根据url判断是播放本地视频还是流视频，播放流视频时候先根据url路劲到本地文件查找是否有已经缓存好的视频文件，有缓存直接播放缓存视频.
缓存是基于AVAssetResourceLoader中间代理对象实现边播变下，抽象出两个类KEPLoaderURLConnection和KEPVideoRequestTask。类KEPLoaderURLConnection是AVAssetResourceLoader的代理对象，实现了AVAssetResourceLoaderDelegate代理方法。KEPVideoRequestTask是执行下载任务的抽象类，目前使用NSUrlConnection来实现。





KEPVideoPlayer
视频播放抽象类
KEPLoaderURLConnection
视频变下边播中间代理类
KEPVideoRequestTask
视频下载实现类
KEPVideoPlayPreView
timeline播放视频视图类
KEPVideoPlayerFullScreenPreView
全屏预览播放视频类


备注
需要注意的地方
* 选填。需要补充的点，遇到的坑之类的。
切换摄像头等操作必须要锁住当前session，在beginConfiguration/commitconfiguration进行
不要在session start之后才设置session相关输入输出配置，会出现闪烁     
不要设置AVAssetWriter实例movieFragmentInterval的属性，大坑(原因没有定位到，可能是设置帧率最小间隔和buffer返回时间不一致导致的)，会出现音频和视频随机性appendSampleBuffer失败，这个坑微信小视频有提到过，在开发过程中掉在这个坑里花费时间最长
麦克风权限关闭如果在代理里面CFRetain了buffer，需要手动CFRelease掉
在timeline播放视频，涉及到cell复用，创建和销毁AVPlayer，不能在当前线程操作，会阻塞当前线程，使用loadValuesAsynchronouslyForKeys异步加载AVURLAsset
在还没有切换到视频录制模块时候，exporter对象为空，需要在didOutputSampleBuffer代理方法CFRelease已经持有的buffer，否则切换到视频录制模块不能录制
AVAssetExportSession导出视频时候设置AVAssetExportPresetMediumQuality得到的分辨率是480*480，会模糊，应该设置AVAssetExportPresetHighestQuality
附录
* 选填。其他参考的文档，链接等。
AVFoundtion
视频压缩方案
在 iOS 上捕获视频
AVFoundation相关类的分析
