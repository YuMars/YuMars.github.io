---
title: "SDWebImage底层代码逻辑"
date: 2024-07-06T14:26:40+08:00
draft: false
---


[TOC]

# SDWebImage

# 什么是SDWebImage
SDWebImage提供了支持缓存的异步图像下载，为UIImageView、UIButton、MKAnnotationView等控件添加了分类，是一个非常方便使用的网络加载图的第三方库。支持（JPEG、PNG、TIFF、BMP、包括GIF / APNG动画图像），甚至是iOS11之后到HEIC格式，iOS14之后的WebP格式，还有如BPG、AVIF、PDF、SVG等

# SDWebImage的作用
## 主要提供的功能有：
1. 图片的缓存管理
2. 异步下载图片
3. 自动缓存过期处理的异步内存+磁盘图像缓存
4 .背景图像解压缩以避免帧率下降
5. 渐进式图像加载(包括动画图像,比如在Web浏览器中显示的GIF)
6. 缩略图图像解码以节省CPU和内存中的大图像 
7. 可扩展的图像编码器以支持大量图像格式,如WebP 
8. 在图像下载后可自定义变换并且组合变换
9. 可自定义和多个加载器系统以扩展功能,如照片库 


![待补充图1](https://www.baidu.com/logo.png)

## 所有功能解析
上图是总体类图，包含了所有用到的类和关联

### 各种分类：
UIButton+WebCache：为UIButton类添加加载图片的方法。
MKAnnotationView+WebCache：为MKAnnotationView类添加各种加载图片的方法。
UIImageView+WebCache：为UIImageView类添加加载图片的方法。
UIImageView+HighlightedWebCache：为UIImageView类添加高亮状态下加载图片的方法。

###    工具类：
NSData+ImageContentType：根据图片数据获取图片的类型，比如GIF、PNG等。
UIImage+MultiFormat：根据UIImage的data生成指定格式的UIImage。
UIImage+GIF：判断一张图是否为GIF。
SDWebImageCompat：根据屏幕的分辨倍数成倍放大或者缩小图片的大小。
SDImageCacheConfig：图片缓存策略记录。比如是否解压缩、是否允许iCloud、是否允许内存缓存、缓存时间等。
SDWebImageCodersManager：编码解码管理器，处理多个图片编码解码任务，编码器是一个优先队列，这意味着后面添加的编码器将具有最高优先级。

### 核心类：
UIView+WebCache：所有的UIView及其子类都会调用这个分类的方法来完成图片加载的处理，同时通过UIView+WebCacheOperation分类来管理请求的取消和记录工作。
SDImageCache：负责SDWebImage的整个缓存工作，是一个单例对象。缓存路径处理、缓存名字处理、管理内存缓存和磁盘缓存的创建和删除、根据指定key获取图片、存入图片的处理、根据缓存的创建和修改日期来删除缓存等。
SDWebImageManager：拥有一个SDImageCache和SDWebImageDownloader属性，分别用于图片的缓存和加载处理。为UIView及其子类提供了加载图片的统一接口。
SDWebImageDownloader：图片下载中心，管理下载队列。
SDWebImageDownloaderOperation：用于下载图片，管理NSURLRequest对象请求头的封装、缓存、cookie的设置、加载选项的处理等。

### 图片设置流程
![待补充图3](https://www.baidu.com/logo.png)
是SDWebImage的调用属性、方法流程图，暴露了有哪些可以使用的加载配置和回调代理。

1.SDWeblmageOptions：是加载图片过程的配置参数，根据参数的不同可以获取做重试、低延时加载、渐进式加载、强制刷新缓存、后台继续加载、处理Cookie、SSL证书信任、下载队列优先级、错误展位图、转换动画、大尺寸图片兼容、缓存配置等等
2.SDWeblmageManager：也可以直接从缓存加载图片，解码图片
3.SDWebImagePrefetcher：预加载图片的一些操作，下载过程处在一个比较低的优先级。

### 图片加载流程
![待补充图2](https://www.baidu.com/logo.png)
上图是SDWebImage图片加载的流程图，实现了图片的加载、数据处理、图片缓存等一系列工作

1. 对象调用暴露的接口方法sd_setImageWithURL()时，会再调用setImageWithURL:placeholderImage:options:方法，先把占位图placeholderImage显示，然后SDWebImageManager根据URL开始处理图片。
2. SDImageCache类先从内存缓存查找是否有图片缓存，如果内存中已经有图片缓存，则直接回调到前端进行图片的显示。
3. 如果内存缓存中没有，则生成NSInvocationOperation添加到队列开始从硬盘中查找图片是否已经缓存。根据url为key在硬盘缓存目录下尝试读取图片文件，这一步是在NSOperation下进行的操作，所以需要回到主线程进行查找结果的回调。如果从硬盘读取到了图片，则将图片添加到内存缓存中，然后再回调到前端进行图片的显示。如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，则需要下载图片。
4. 共享或重新生成一个下载器SDWebImageDownloader开始下载图片。图片的下载由NSURLConnection来处理，实现相关delegate来判断的下载状态：下载中、下载完成和下载失败。
5. 图片数据下载完成之后，交给SDWebImageDecoder类做图片解码处理，图片的解码处理在NSOperationQueue完成，不会阻塞主线程。在图片解码完成后，会回调给SDWebImageDownloader，然后回调给SDWebImageManager告知图片下载完成，通知所有的downloadDelegates下载完成，回调给需要的地方显示图片。
6. 最后将图片通过SDImageCache类，同时保存到内存缓存和硬盘缓存中。写文件到硬盘的过程也在以单独NSInvocationOperation完成，避免阻塞主线程。

一般问SDWebImage的加载流程，普遍是知道SDWebImage的加载流程是先从缓存中取图片，再从磁盘中取图片，最后再做下载。下载过程是如何保存到缓存中，保存在磁盘中，磁盘中以什么方式存储，什么方式命名，缓存中以什么方式存储，以什么方式命名，缓存满了如何处理？怎么管理缓存，怎么管理磁盘，什么时候删除缓存数据，什么时候删除磁盘数据。如何手动删除缓存中数据、如何手动删除磁盘中数据。带着问题来从头开始看。

#### 入口函数
  sd_setImageWithURL所有类似方法的入口函数都是通过sd_internalSetImageWithURL:placeholderImage:optionscontext:setImageBlock:progress:completed:
  
#### 参数
url:图片下载网络地址。大部分用的url的地方前置会判断url为空的情况下直接return，并且会在下载动作前判断传入类型是NSString下转成NSURL（主要是类型强转的场景）

placeholderImage：展位图，多种情况下会使用，未加载图片前占位图、加载失败后占位图，通过SDWebImageOptions控制

SDWebImageOptions：缓存类型，有23种，用于处理是否缓存，怎么加载缓存（内存缓存memoryCache、磁盘缓存dickCache），强制刷新、失败重试等等情况，可以叠加使用
```
typedef NS_OPTIONS(NSUInteger, SDWebImageOptions) {
    /**
     * 当URL失败下载时,该URL会被列入黑名单,以便库不会一直尝试。 此标志禁用此黑名单。
     */
    SDWebImageRetryFailed = 1 << 0,
    
    /**
     * 图像下载在UI交互期间开始,此标志禁用此功能, 导致UIScrollView减速时的延迟下载。
     */
    SDWebImageLowPriority = 1 << 1,
    
    /**
     * 此标志启用渐进式下载,图像在下载过程中逐渐显示,就像浏览器所做的那样。 默认情况下,仅在完全下载后显示图像。
     */
    SDWebImageProgressiveLoad = 1 << 2,
    
    /**
     * 只有在无法使您的URL静态并嵌入缓存清除参数的情况下,才使用此标志。
     */
    SDWebImageRefreshCached = 1 << 3,
    
    /**
     * 在iOS 4+中,如果应用程序进入后台,继续下载图像。这是通过请求系统提供额外的后台时间来完成请求来实现的。如果后台任务过期,操作将被取消
     */
    SDWebImageContinueInBackground = 1 << 4,
    
    /**
     * 通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES来处理NSHTTPCookieStore中的cookie;
     */
    SDWebImageHandleCookies = 1 << 5,
    
    /**
     * 启用以允许不受信任的SSL证书。
     */
    SDWebImageAllowInvalidSSLCertificates = 1 << 6,
    
    /**
     * 默认情况下,图像按照它们排队的顺序加载。此标志将它们移动到队列的前面。
     */
    SDWebImageHighPriority = 1 << 7,
    
    /**
     * 默认情况下,加载图像时显示占位符图像。此标志将推迟加载
     * 直到图像完成加载之后显示占位符图像。
     */
    SDWebImageDelayPlaceholder = 1 << 8,
    
    /**
     * 我们通常不对动画图像应用变换,因为大多数变换器无法处理动画图像。使用此标志无论如何对它们进行变换。
     */
    SDWebImageTransformAnimatedImage = 1 << 9,
    
    /**
     * 默认情况下,图像在下载后添加到图像视图。但在某些情况下,我们希望在设置图像之前拥有手部(例如应用过滤器或添加交叉渐变动画)。如果要在成功时手动设置图像,请使用此标志
     */
    SDWebImageAvoidAutoSetImage = 1 << 10,
    
    /**
     * 默认情况下,图像按照其原始大小解码。
     * 此标志将缩小图像到设备受限内存兼容的大小。
     * 要控制限制内存字节,请检查`SDImageCoderHelper.defaultScaleDownLimitBytes`(在iOS上默认为60MB)
     * 这实际上对应于v5.5.0中使用的上下文选项`.imageThumbnailPixelSize`(在iOS上默认为(3966,3966))。以前不会。
     */
    SDWebImageScaleDownLargeImages = 1 << 11,
    
    /**
     * 默认情况下,当图像已经缓存在内存中时,我们不查询图像数据。此掩码可以强制同时查询图像数据。但是,除非指定`SDWebImageQueryMemoryDataSync`,否则此查询是异步的。
     */
    SDWebImageQueryMemoryData = 1 << 12,
    
    /**
     * 默认情况下,当您仅指定`SDWebImageQueryMemoryData`时,我们异步查询内存中的图像数据。结合此掩码也可以同步查询内存中的图像数据。
     */
    SDWebImageQueryMemoryDataSync = 1 << 13,
    
    /**
     * 默认情况下,当内存缓存未命中时,我们异步查询磁盘缓存。此掩码可以强制在内存缓存未命中时同步查询磁盘缓存。
     */
    SDWebImageQueryDiskDataSync = 1 << 14,
    
    /**
     * 默认情况下,当缓存未命中时,图像将从加载程序加载。此标志可阻止仅从缓存加载。
     */
    SDWebImageFromCacheOnly = 1 << 15,
    
    /**
     * 默认情况下,我们在图像从加载程序加载之前查询缓存。此标志可阻止仅从加载程序加载。
     */
    SDWebImageFromLoaderOnly = 1 << 16,
    
    /**
     *  默认情况下,当您使用`SDWebImageTransition`在图像加载完成后进行某些视图过渡时,此过渡仅适用于来自网络或磁盘缓存查询的异步回调产生的图像。
     * 此掩码可以强制对任何情况(如内存缓存查询或同步磁盘缓存查询)应用视图过渡。
     */
    SDWebImageForceTransition = 1 << 17,
    
    /**
     * 默认情况下,我们将在缓存查询和网络下载期间在后台解码图像。这可以帮助提高性能,因为在屏幕上渲染图像时,首先需要解码。但是,这发生在Core Animation的主队列上。
     * 然而,此过程也可能增加内存使用量。如果由于过度内存消耗而遇到问题,此标志可以防止解码图像。
     */
    SDWebImageAvoidDecodeImage = 1 << 18,
    
    /**
     * 默认情况下,我们解码动画图像。此标志可以强制仅解码第一个帧并生成静态图像。
     */
    SDWebImageDecodeFirstFrameOnly = 1 << 19,
    
    /**
     * 默认情况下,对于`SDAnimatedImage`,我们在渲染期间解码动画图像帧以减少内存使用量。但是,您可以指定将所有帧预加载到内存中,以减少动画图像由许多imageViews共享时的CPU使用量。
     * 这实际上会触发后台队列(磁盘缓存和下载)中的`preloadAllAnimatedImageFrames`。
     */
    SDWebImagePreloadAllFrames = 1 << 20,
    
    /**
     * 默认情况下,当您使用`SDWebImageContextAnimatedImageClass`上下文选项(如使用`SDAnimatedImageView`,旨在使用`SDAnimatedImage`)时,我们可能仍然使用`UIImage`当内存缓存命中时,或图像解码器不可用来生成一个完全匹配您的自定义类作为后备解决方案.使用此选项,可以确保我们始终使用您提供的类回调图像。如果无法生成,将使用代码为`SDWebImageErrorBadImageData`的错误。
     */
    SDWebImageMatchAnimatedImageClass = 1 << 21,
    
    /**
     * 默认情况下,当我们从网络加载图像时,图像将被写入缓存(内存和磁盘,由您的`storeCacheType`上下文选项控制)。
     * 这可能是一个异步操作,最终的`SDInternalCompletionBlock`回调不能保证磁盘缓存写入已完成,并可能导致逻辑错误。 (例如,您只在完成块中修改磁盘数据,但是磁盘缓存尚未准备好)
     * 如果您需要在完成块中处理磁盘缓存,应使用此选项以确保在回调时磁盘缓存已经写入。
     */
    SDWebImageWaitStoreCache = 1 << 22,
    
    /**
     * 我们通常不对矢量图像应用变换,因为矢量图像支持动态更改为任何大小,对固定大小进行光栅化会损失细节。要修改矢量图像,您可以在运行时处理矢量数据(如修改PDF标记/SVG元素)。
     * 使用此标志无论如何对它们进行变换。
     */
    SDWebImageTransformVectorImage = 1 << 23
}
```

SDWebImageContext:就是NSDictionary，方便自定义，在整个加载流程中一直优先以SDWebImageContext传入的内容优先，没有则通过SDWebImage内部提供的属性替换

```
SDWebImageContextOption const SDWebImageContextSetImageOperationKey = @"setImageOperationKey"; // 通过validOperationKey取消正在运行的任务
SDWebImageContextOption const SDWebImageContextCustomManager = @"customManager"; // 初始化SDWebImageManager
SDWebImageContextOption const SDWebImageContextCallbackQueue = @"callbackQueue"; // Cache、Manager、Loader的回调，可以控制回调，不如不在主队列回调
SDWebImageContextOption const SDWebImageContextImageCache = @"imageCache";// 如果您想在图像加载期间指定自定义缓存,则无需重新创建具有缓存的假SDWebImageManager实例
SDWebImageContextOption const SDWebImageContextImageLoader = @"imageLoader"; // 实现自定义图片加载器
SDWebImageContextOption const SDWebImageContextImageCoder = @"imageCoder"; // 实现自定义图片解码器
SDWebImageContextOption const SDWebImageContextImageTransformer = @"imageTransformer"; // 自定义图片加载出来的效果动画
SDWebImageContextOption const SDWebImageContextImageDecodeOptions = @"imageDecodeOptions"; // 解码相关选项
SDWebImageContextOption const SDWebImageContextImageScaleFactor = @"imageScaleFactor"; // 解码的时候自定义比例图像缩放比 2x,3x，默认是屏幕的缩放
SDWebImageContextOption const SDWebImageContextImagePreserveAspectRatio = @"imagePreserveAspectRatio";// 控制在为图像生成缩略图时是否要强制缩略图具有与原始图像相同的纵横比
SDWebImageContextOption const SDWebImageContextImageThumbnailPixelSize = @"imageThumbnailPixelSize"; // 加载图像时指定生成缩略图的最大像素大小。将解码并缓存缩略图,并在回调中返回缩略图,而原始数据将保留在磁盘上。
SDWebImageContextOption const SDWebImageContextImageTypeIdentifierHint = @"imageTypeIdentifierHint";// 为图像提供文件扩展名或UTI提示,以帮助图像解码器选择正确的图像格式解析器。
SDWebImageContextOption const SDWebImageContextImageEncodeOptions = @"imageEncodeOptions"; // 通过编码选项自定义图像数据的编码方式。
SDWebImageContextOption const SDWebImageContextQueryCacheType = @"queryCacheType";// 指定SDWebImage从哪个缓存(内存缓存,磁盘缓存或两者)加载图像数据。
SDWebImageContextOption const SDWebImageContextStoreCacheType = @"storeCacheType"; // 指定下载的图像数据应存储在哪个SDWebImage缓存(内存,磁盘或两者)中。
SDWebImageContextOption const SDWebImageContextOriginalQueryCacheType = @"originalQueryCacheType"; // 指定在使用图像转换器时,SDWebImage应首先从哪个缓存(内存,磁盘或两者)中加载原始图像数据。
SDWebImageContextOption const SDWebImageContextOriginalStoreCacheType = @"originalStoreCacheType"; // 指定在使用图像转换器时,SDWebImage应首先将原始图像数据存储在哪个缓存(内存,磁盘或两者)中。
SDWebImageContextOption const SDWebImageContextOriginalImageCache = @"originalImageCache"; // 自定义缓存来控制原始图像数据的加载和存储。转换后的图像将继续使用SDWebImage的默认内存和磁盘缓存。
SDWebImageContextOption const SDWebImageContextAnimatedImageClass = @"animatedImageClass"; // 自定义的动画图像类以提高性能。SDWebImage会尝试使用此类来解析动画图像数据,而不是默认的UIImage。
SDWebImageContextOption const SDWebImageContextDownloadRequestModifier = @"downloadRequestModifier"; // 自定义的请求修改器来操作SDWebImage进行的图像下载请求。
SDWebImageContextOption const SDWebImageContextDownloadResponseModifier = @"downloadResponseModifier"; // 自定义的响应修改器来操作SDWebImage执行的图像下载响应。
SDWebImageContextOption const SDWebImageContextDownloadDecryptor = @"downloadDecryptor"; // 自定义的数据解密器来解密SDWebImage下载的图像数据。
SDWebImageContextOption const SDWebImageContextCacheKeyFilter = @"cacheKeyFilter"; // 自定义的缓存键过滤器,以生成SDWebImage用于图像缓存的密钥。
SDWebImageContextOption const SDWebImageContextCacheSerializer = @"cacheSerializer"; // 自定义的缓存序列化器来序列化/反序列化SDWebImage用于磁盘缓存的图像数据。
```

SDSetImageBlock：设置图像时的回调

SDImageLoaderProgressBlock：图片加载进度回调

SDInternalCompletionBlock：图片加载完成回调

从SDWebImageContext可以看出对应SDWebImage有哪些主要的功能。其对应的类也列出
1. 取消任务：SDWebImageOperation
2. 回调任务控制 SDWebImageDownloaderOperation（下载任务） SDWebImageManager（进度、完成任务） SDImageCache（缓存 ）SDWebImagePrefetcher（预加载任务）
3. 缓存图片（磁盘、内存，缓存的位置） ：SDImageCache
4. 图片加载（磁盘、缓存、请求数据、响应数据）SDImageLoader
5. 图片解码（解码后图片缩放比、宽高比、文件信息）SDImageCoder
6. 图片加载后显示动画（动画方式）SDImageTransformer
7. 图片编码（需要将图片缓存，所以需要重新序列化图片数据）
8. 图片下载：SDWebImageDownloader


```
- (nullable id<SDWebImageOperation>)sd_internalSetImageWithURL:(nullable NSURL *)url
                                              placeholderImage:(nullable UIImage *)placeholder
                                                       options:(SDWebImageOptions)options
                                                       context:(nullable SDWebImageContext *)context
                                                 setImageBlock:(nullable SDSetImageBlock)setImageBlock
                                                      progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                                     completed:(nullable SDInternalCompletionBlock)completedBlock {
    if (context) {
        // copy to avoid mutable object
        context = [context copy];
    } else {
        context = [NSDictionary dictionary];
    }
    
    // 先取了context里面的配置参数，context没有，则从NSStringFromClass([self class])取类本身，并且给context赋值{@"setImageOperationKey":validOperationKey}
    NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
    if (!validOperationKey) {
        // pass through the operation key to downstream, which can used for tracing operation or image view class
        validOperationKey = NSStringFromClass([self class]);
        SDWebImageMutableContext *mutableContext = [context mutableCopy];
        mutableContext[SDWebImageContextSetImageOperationKey] = validOperationKey;
        context = [mutableContext copy];
    }
    self.sd_latestOperationKey = validOperationKey;
    
    // 从sd_operationDictionary中取出SDOperationsDictionary对应的operation，并且取消
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    self.sd_imageURL = url;
    
    SDWebImageManager *manager = context[SDWebImageContextCustomManager];
    if (!manager) {
        manager = [SDWebImageManager sharedManager];
    } else {
        // 防止传入的context循环引用SDWebImageManager
        // remove this manager to avoid retain cycle (manger -> loader -> operation -> context -> manager)
        SDWebImageMutableContext *mutableContext = [context mutableCopy];
        mutableContext[SDWebImageContextCustomManager] = nil;
        context = [mutableContext copy];
    }
    
    // 弱引用Cache,当内存警告时候缓存memoryCahce中的数据被清除，一些图片就需要重新下载，用了shouldUseWeakCache后清理缓存后也可以从NSMapTable类型的weakCache表中获取数据
    BOOL shouldUseWeakCache = NO;
    if ([manager.imageCache isKindOfClass:SDImageCache.class]) {
        shouldUseWeakCache = ((SDImageCache *)manager.imageCache).config.shouldUseWeakMemoryCache;
    }
    if (!(options & SDWebImageDelayPlaceholder)) { // 没有设置延迟placeHolder（加载过程取消或者失败的情况）
        if (shouldUseWeakCache) {
            NSString *key = [manager cacheKeyForURL:url context:context];
            // call memory cache to trigger weak cache sync logic, ignore the return value and go on normal query
            // this unfortunately will cause twice memory cache query, but it's fast enough
            // in the future the weak cache feature may be re-design or removed
            [((SDImageCache *)manager.imageCache) imageFromMemoryCacheForKey:key];
        }
        dispatch_main_async_safe(^{// 判断了当前线程是不是在主线程，确保在主线程执行
            // 设置placehodler
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:SDImageCacheTypeNone imageURL:url];
        });
    }
    
    // 遵循的（可取消）协议的对象-下载功能
    id <SDWebImageOperation> operation = nil;
    
    // 1.有URL开始下载，开始加载动画、并且同步进度
    // 2.没URL，停止加载动画、回调进度、同时报错
    if (url) {
        // reset the progress // 设置加载进度
        NSProgress *imageProgress = objc_getAssociatedObject(self, @selector(sd_imageProgress));
        if (imageProgress) {
            imageProgress.totalUnitCount = 0;
            imageProgress.completedUnitCount = 0;
        }
        
        // check and start image indicator
        // 图片加载转菊花，有设置就开始动画，没有设置就return
        [self sd_startImageIndicator];
        id<SDWebImageIndicator> imageIndicator = self.sd_imageIndicator;
        
        // 加载进度回调
        // 对外部的imageProgress设置:已经加载的数据量和还需要的数据量
        // 同时对转菊花的进行进度更新
        SDImageLoaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            if (imageProgress) {
                imageProgress.totalUnitCount = expectedSize;
                imageProgress.completedUnitCount = receivedSize;
            }
            
            if ([imageIndicator respondsToSelector:@selector(updateIndicatorProgress:)]) {
                double progress = 0;
                if (expectedSize != 0) {
                    progress = (double)receivedSize / expectedSize;
                }
                progress = MAX(MIN(progress, 1), 0); // 0.0 - 1.0
                dispatch_async(dispatch_get_main_queue(), ^{
                    [imageIndicator updateIndicatorProgress:progress];
                });
            }
            
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
        @weakify(self);
        
        operation = [manager loadImageWithURL:url options:options context:context progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            
            // 加载完成后操作、设置加载动画、设置图片转场效果、设置最终图片
            
            @strongify(self);
            if (!self) { return; }
            
            // 纠错，如果已经完成并且没有报错，将已下载总数据量和总数据量标记成完成状态(可能是那种数据量很少，直接下载完成无进度回调的情况)
            if (imageProgress && finished && !error && imageProgress.totalUnitCount == 0 && imageProgress.completedUnitCount == 0) {
                imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
            }
            
            // 结束加载菊花动画
            if (finished) {
                [self sd_stopImageIndicator];
            }
            
            // SDWebImageAvoidAutoSetImage手动设置加载完成后操作
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            // SDWebImageDelayPlaceholder手动设置加载成功后动作
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            
            // 设置SDWebImageAvoidAutoSetImage和SDWebImageDelayPlaceholder的回调
            SDWebImageNoParamsBlock callCompletedBlockClosure = ^{
                if (!self) { return; }
                if (!shouldNotSetImage) {
                    [self sd_setNeedsLayout];
                }
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, data, error, cacheType, finished, url);
                }
            };
            
            // 有图片，但是需要手动设置图片
            // 无图片，但是placeHolder没设置
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClosure);
                return;
            }
            
            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // 有图，自动设置图片
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // 无图，设置了PlaceHolder
                targetImage = placeholder;
                targetData = nil;
            }
            
            // 加载成功后动画专场效果

            SDWebImageTransition *transition = nil;
            BOOL shouldUseTransition = NO;
            if (options & SDWebImageForceTransition) {
                // 强制转场效果
                shouldUseTransition = YES;
            } else if (cacheType == SDImageCacheTypeNone) {
                // 没有缓存，需要转场效果
                shouldUseTransition = YES;
            } else {
                // 从缓存中取的，不需要转场
                // 从磁盘中取的，如果是缓存同步或者磁盘同步，不需要转场，否则需要转场
                if (cacheType == SDImageCacheTypeMemory) {
                    shouldUseTransition = NO;
                } else if (cacheType == SDImageCacheTypeDisk) {
                    if (options & SDWebImageQueryMemoryDataSync || options & SDWebImageQueryDiskDataSync) {
                        shouldUseTransition = NO;
                    } else {
                        shouldUseTransition = YES;
                    }
                } else {
                    // 其他情况不需要转场
                    shouldUseTransition = NO;
                }
            }
            if (finished && shouldUseTransition) {
                transition = self.sd_imageTransition;
            }

            dispatch_main_async_safe(^{
                [self sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];

                // 加载成功回调
                callCompletedBlockClosure();
            });
        }];
        // 设置operation和validOperationKey关联、用于取消某次加载图片
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } else {
        // 停止图片加载动画
        [self sd_stopImageIndicator];
        if (completedBlock) {
            dispatch_main_async_safe(^{
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
                completedBlock(nil, nil, error, SDImageCacheTypeNone, YES, url);
            });
        }
    }
    
    return operation;
}
```

### 入口函数里面主要的功能：
1. 如果有需要需要的任务，则取消已经开始的任务
2. 判断是否使用了弱引用，用于内存警告的情况下，用户还能显示图片
3. 根据SDWebImageDelayPlaceholder，不是就在加载完成显示占位图，否则就先显示占位图
4. 将当前任务加到SDOperationsDictionary中
5. 根据有无加载动画，显示加载动画
6. 无URL的、停止动画，报错回调
7. 有URL的、初始化加载进度，重复第4步，开始加载图片
8. 图片加载成功后回调
  8.1 同步进度imageProgress
  8.2 结束加载动画
  8.3 根据SDWebImageAvoidAutoSetImage判断，是否需要手动设置图片
  8.4 根据options判断是否需要做转场动画
  8.5 回调completedBlock(image, data, error, cacheType, finished, url)

# 加载图片SDWebImageManager
加载图片调用SDWebImageManager的loadImageWithURL:options:context:progress:completed
```
- (SDWebImageCombinedOperation *)loadImageWithURL:(nullable NSURL *)url options:(SDWebImageOptions)options context:(nullable SDWebImageContext *)context progress:(nullable SDImageLoaderProgressBlock)progressBlock completed:(nonnull SDInternalCompletionBlock)completedBlock {
    // Invoking this method without a completedBlock is pointless
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    // 常见错误适配
    // 强制转换NSString成NSURL
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // 保护：做其他类型判断
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

    SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    operation.manager = self;

    BOOL isFailedUrl = NO;
    if (url) {
        // 判断是已经尝试过失败的URL集合里的数据，下一步处理中直接报错，并且结束操作
        SD_LOCK(_failedURLsLock);
        isFailedUrl = [self.failedURLs containsObject:url];
        SD_UNLOCK(_failedURLsLock);
    }
    
    // Preprocess the options and context arg to decide the final the result for manager
    // 预处理选项和上下文参数,以决定管理器的最终结果。
    // 1.Transformer，图片动画
    // 2.cacheKeyFilter 缓存key过滤
    // 3.cacheSerializer 将解码的图像、源下载的数据转换为并存储到磁盘缓存，用于已经加载到的大数据图片比较慢，可以转换成其他格式存储到磁盘缓存中
    // 4.如果有全局控制options处理操作，按照全局控制options处理操作优先，否则就用设置的操作进行
    SDWebImageOptionsResult *result = [self processedResultForURL:url options:options context:context];

    // 如果URL空，或者options是不需要重试的失败url，直接报错，返回对应的操作，失败的结果
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        NSString *description = isFailedUrl ? @"Image url is blacklisted" : @"Image url is nil";
        NSInteger code = isFailedUrl ? SDWebImageErrorBlackListed : SDWebImageErrorInvalidURL;
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:code userInfo:@{NSLocalizedDescriptionKey : description}] queue:result.context[SDWebImageContextCallbackQueue] url:url];
        return operation;
    }

    SD_LOCK(_runningOperationsLock);
    // 下载操作中的集合添加新的操作任务
    [self.runningOperations addObject:operation];
    SD_UNLOCK(_runningOperationsLock);
    
    // 从缓存加载图像的入口
    // 1.没有图片转场的步骤->从缓存查询图像->下载数据和图像->将图像存储在缓存中
    
    // 2.有图片转场的步骤->从缓存查询转换后的图像->从缓存查询原始图像->下载数据和图像->在CPU上进行变换->将原始图像存储在缓存中->将转换后的图像存储在缓存中
    
    [self callCacheProcessForOperation:operation url:url options:result.options context:result.context progress:progressBlock completed:completedBlock];

    return operation;
}
```

### SDWebImageManager执行的代码逻辑：
1. url纠错
2. 生成return的SDWebImageCombinedOperation对象
3. 判断当前图片URL是否在失败集合failedURLs中，根据options判断是否需要重新加载失败的url，还是结束流程并且返回错误信息
4. 将当前SDWebImageCombinedOperation加到runningOperations集合中，自动判断runningOperations中有否对象，自动执行后续的任务
5. 调用查询普通缓存方法callCacheProcessForOperation查看是否需要使用缓存中的内容


### 缓存查询过程
查询缓存callCacheProcessForOperation:url:options:context:progress:completed
1.没有图片转场的步骤->从缓存查询图像->下载数据和图像->将图像存储在缓存中
2.有图片转场的步骤->从缓存查询转换后的图像->从缓存查询原始图像->下载数据和图像->在CPU上进行变换->将原始图像存储在缓存中->将转换后的图像存储在缓存中
```
- (void)callCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                 url:(nonnull NSURL *)url
                             options:(SDWebImageOptions)options
                             context:(nullable SDWebImageContext *)context
                            progress:(nullable SDImageLoaderProgressBlock)progressBlock
                           completed:(nullable SDInternalCompletionBlock)completedBlock {
    // Grab the image cache to use
    //
    id<SDImageCache> imageCache = context[SDWebImageContextImageCache];
    if (!imageCache) {
        imageCache = self.imageCache;
    }
    
    // 默认支持从缓存、磁盘取
    SDImageCacheType queryCacheType = SDImageCacheTypeAll;
    if (context[SDWebImageContextQueryCacheType]) {
        queryCacheType = [context[SDWebImageContextQueryCacheType] integerValue];
    }
    
    // 查询缓存
    BOOL shouldQueryCache = !SD_OPTIONS_CONTAINS(options, SDWebImageFromLoaderOnly);
    if (shouldQueryCache) { // 取缓存过程
        // 根据URL返回缓存的key
        // 1.优先从cacheKeyFilter取key
        // 2.没有保底用url.absoluteString作为key
        // 3.取地址和缩略图尺寸宽高比的作为key
        // 4.取transformer作为作为key
        NSString *key = [self cacheKeyForURL:url context:context];
        @weakify(operation);
        operation.cacheOperation = [imageCache queryImageForKey:key options:options context:context cacheType:queryCacheType completion:^(UIImage * _Nullable cachedImage, NSData * _Nullable cachedData, SDImageCacheType cacheType) {
            @strongify(operation);
            if (!operation || operation.isCancelled) { // 取消组合操作 -> 直接结束，并且报错 -> 删除running的queue
                [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:@{NSLocalizedDescriptionKey : @"Operation cancelled by user during querying the cache"}] queue:context[SDWebImageContextCallbackQueue] url:url];
                [self safelyRemoveOperationFromRunning:operation];
                return;
            } else if (!cachedImage) { // 未取到缓存图片(并且已经有缓存key，说明在解码过程中或者在图形转换过程) -> 走下载流程
                NSString *originKey = [self originalCacheKeyForURL:url context:context];
                BOOL mayInOriginalCache = ![key isEqualToString:originKey];
                // 有机会查询原始缓存而不是下载,然后应用变换
                // 缩略图解码是在SDImageCache的解码部分完成的,不需要变换的后期处理
                if (mayInOriginalCache) {
                    // 如果原始缓存中有，则查询原始缓存
                    [self callOriginalCacheProcessForOperation:operation url:url options:options context:context progress:progressBlock completed:completedBlock];
                    return;
                }
            }
            // 下载流程
            [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:cachedImage cachedData:cachedData cacheType:cacheType progress:progressBlock completed:completedBlock];
        }];
    } else { // 下载过程
        // Continue download process
        [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
    }
}
```

### 缓存中查询主要做了：
1. 判断查询类型，SDWebImageFromLoaderOnly默认有缓存仅从缓存中查询
    1.1 缓存查询后判断当前SDWebImageCombinedOperation是否已经取消，如果取消，回调失败结果和原因
    1.2 如果未查询到缓存，判断是否已经有生成过缓存key，如果有原始缓存key，调用查询原始缓存流程
2. 不查询缓存则走下载流程


### 查询原始缓存
callOriginalCacheProcessForOperation:url:options:context:progress:completed
```
// 查询原始缓存过程
- (void)callOriginalCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                         url:(nonnull NSURL *)url
                                     options:(SDWebImageOptions)options
                                     context:(nullable SDWebImageContext *)context
                                    progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    // 获取要使用的图像缓存,首先选择独立的原始缓存
    id<SDImageCache> imageCache = context[SDWebImageContextOriginalImageCache];
    if (!imageCache) {
        // 如果没有独立缓存可用,使用默认缓存
        imageCache = context[SDWebImageContextImageCache];
        if (!imageCache) {
            imageCache = self.imageCache;
        }
    }
    
    // 默认从磁盘查询
    SDImageCacheType originalQueryCacheType = SDImageCacheTypeDisk;
    if (context[SDWebImageContextOriginalQueryCacheType]) {
        originalQueryCacheType = [context[SDWebImageContextOriginalQueryCacheType] integerValue];
    }
    
    // 确认是否需要需要查询原始缓存
    BOOL shouldQueryOriginalCache = (originalQueryCacheType != SDImageCacheTypeNone);
    if (shouldQueryOriginalCache) {
        // 获取没有变换器的原始缓存键生成
        NSString *key = [self originalCacheKeyForURL:url context:context];
        @weakify(operation);
        operation.cacheOperation = [imageCache queryImageForKey:key options:options context:context cacheType:originalQueryCacheType completion:^(UIImage * _Nullable cachedImage, NSData * _Nullable cachedData, SDImageCacheType cacheType) {
            @strongify(operation);
            if (!operation || operation.isCancelled) {// 已经取消，直接结束并且报错
                
                [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:@{NSLocalizedDescriptionKey : @"Operation cancelled by user during querying the cache"}] queue:context[SDWebImageContextCallbackQueue] url:url];
                [self safelyRemoveOperationFromRunning:operation];
                return;
            } else if (!cachedImage) {
                // 无缓存，走下载流程
                [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
                return;
            }

            // 跳过下载并继续变换过程,暂时忽略.refreshCached选项
            [self callTransformProcessForOperation:operation url:url options:options context:context originalImage:cachedImage originalData:cachedData cacheType:cacheType finished:YES completed:completedBlock];
            
            [self safelyRemoveOperationFromRunning:operation];
        }];
    } else {
        // 继续下载流程
        [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
    }
}
```

###    查询原始缓存主要代码：
1. 获取查询缓存imageCache，判断是否需要查询原始缓存，如果需要则查询缓存
    1.1 通过缓存查询key对应缓存图片cachedImage、缓存数据cachedData、缓存类型SDImageCacheType
    1.2 缓存查询结束回调
    1.3    查询缓存操作取消则直接返回错误callCompletionBlockForOperation:completion:error:queue:url
    1.4 未查询到缓存，执行下载流程callDownloadProcessForOperation:url:options:context:cachedImage:cachedData:cacheType:progress:completed
    1.5 查询到缓存图片，执行转场效果方法callTransformProcessForOperation:url:options:context:originalImage:originalData:originalData:finished:completed
    1.6 将当前SDWebImageCombinedOperation从runningOperations中移除

2. 不需要查询缓存，走下载流程

转场特效方法:callTransformProcessForOperation:url:options:context:originalImage:originalData:originalData:finished:completed

```
// Transform process
- (void)callTransformProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                     url:(nonnull NSURL *)url
                                 options:(SDWebImageOptions)options
                                 context:(SDWebImageContext *)context
                           originalImage:(nullable UIImage *)originalImage
                            originalData:(nullable NSData *)originalData
                               cacheType:(SDImageCacheType)cacheType
                                finished:(BOOL)finished
                               completed:(nullable SDInternalCompletionBlock)completedBlock {
    id<SDImageTransformer> transformer = context[SDWebImageContextImageTransformer];
    if ([transformer isEqual:NSNull.null]) {
        transformer = nil;
    }
    
    // 是否需要转场特效
    BOOL shouldTransformImage = originalImage && transformer;
    shouldTransformImage = shouldTransformImage && (!originalImage.sd_isAnimated || (options & SDWebImageTransformAnimatedImage));
    shouldTransformImage = shouldTransformImage && (!originalImage.sd_isVector || (options & SDWebImageTransformVectorImage));
    
    BOOL isThumbnail = originalImage.sd_isThumbnail;
    NSData *cacheData = originalData;
    UIImage *cacheImage = originalImage;
    if (isThumbnail) { // 缩略图无全图数据和尺寸
        cacheData = nil; // thumbnail don't store full size data
        originalImage = nil; // thumbnail don't have full size image
    }
    
    if (shouldTransformImage) {
        
        NSString *key = [self cacheKeyForURL:url context:context];
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
            // Case that transformer on thumbnail, which this time need full pixel image
            UIImage *transformedImage = [transformer transformedImageWithImage:cacheImage forKey:key];
            // 有转场特效原图用transformedImage原图，否则就用cacheImage缓存图
            if (transformedImage) { // 转场图
                transformedImage.sd_isTransformed = YES;
                [self callStoreOriginCacheProcessForOperation:operation url:url options:options context:context originalImage:originalImage cacheImage:transformedImage originalData:originalData cacheData:nil cacheType:cacheType finished:finished completed:completedBlock];
            } else {
                [self callStoreOriginCacheProcessForOperation:operation url:url options:options context:context originalImage:originalImage cacheImage:cacheImage originalData:originalData cacheData:cacheData cacheType:cacheType finished:finished completed:completedBlock];
            }
        });
    } else {
        [self callStoreOriginCacheProcessForOperation:operation url:url options:options context:context originalImage:originalImage cacheImage:cacheImage originalData:originalData cacheData:cacheData cacheType:cacheType finished:finished completed:completedBlock];
    }
}
```
### 转场特效流程主要代码：
1. 获取SDImageTransformer图片改变配置，判断是否需要转场动画
2. 需要做图片改变
    2.1 生成了改变后的图片，执行保存改变后的图和原始图片到缓存中callStoreOriginCacheProcessForOperation:url:options:context:originalImage:cacheImage:originalData:cacheData:cacheType:finished:completed:
    2.2 未生成改变后的图片，执行保存原始图片到缓存中callStoreOriginCacheProcessForOperation:url:options:context:originalImage:cacheImage:originalData:cacheData:cacheType:finished:completed:
3. 不需要做图片改变，保存原始图片和原始数据

### 存储原始缓存过程
callStoreOriginCacheProcessForOperation:url:options:context:originalImage:cacheImage:originalData:cacheData:cacheType:finished:completed:

```
// 存储原始缓存过程
- (void)callStoreOriginCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                            url:(nonnull NSURL *)url
                                        options:(SDWebImageOptions)options
                                        context:(SDWebImageContext *)context
                                  originalImage:(nullable UIImage *)originalImage
                                     cacheImage:(nullable UIImage *)cacheImage
                                   originalData:(nullable NSData *)originalData
                                      cacheData:(nullable NSData *)cacheData
                                      cacheType:(SDImageCacheType)cacheType
                                       finished:(BOOL)finished
                                      completed:(nullable SDInternalCompletionBlock)completedBlock {
    // 获取要使用的图像缓存,首先选择独立的原始缓存
    id<SDImageCache> imageCache = context[SDWebImageContextOriginalImageCache];
    if (!imageCache) {
        imageCache = context[SDWebImageContextImageCache];
        if (!imageCache) {
            imageCache = self.imageCache;
        }
    }
    
    // 默认存到磁盘
    SDImageCacheType originalStoreCacheType = SDImageCacheTypeDisk;
    if (context[SDWebImageContextOriginalStoreCacheType]) {
        originalStoreCacheType = [context[SDWebImageContextOriginalStoreCacheType] integerValue];
    }
    id<SDWebImageCacheSerializer> cacheSerializer = context[SDWebImageContextCacheSerializer];
    
    // 如果原始cacheType是磁盘,因为我们不需要再次存储原始数据，从原始StoreCacheType中剥离磁盘
    if (cacheType == SDImageCacheTypeDisk) {
        if (originalStoreCacheType == SDImageCacheTypeDisk) originalStoreCacheType = SDImageCacheTypeNone;
        if (originalStoreCacheType == SDImageCacheTypeAll) originalStoreCacheType = SDImageCacheTypeMemory;
    }
    
    // 获取没有变换器的原始缓存键生成
    NSString *key = [self originalCacheKeyForURL:url context:context];
    if (finished && cacheSerializer && (originalStoreCacheType == SDImageCacheTypeDisk || originalStoreCacheType == SDImageCacheTypeAll)) { // 缓存需要序列化（转成Data数据）
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
            // 对图片按照指定的格式jpg/png进行缓存
            NSData *newOriginalData = [cacheSerializer cacheDataWithImage:originalImage originalData:originalData imageURL:url];
            // 存原始数据和图片
            [self storeImage:originalImage imageData:newOriginalData forKey:key options:options context:context imageCache:imageCache cacheType:originalStoreCacheType finished:finished completion:^{
                // 继续存储缓存过程,变换的数据为nil
                [self callStoreCacheProcessForOperation:operation url:url options:options context:context image:cacheImage data:cacheData cacheType:cacheType finished:finished completed:completedBlock];
            }];
        });
    } else {
        // 存原始数据和图片
        [self storeImage:originalImage imageData:originalData forKey:key options:options context:context imageCache:imageCache cacheType:originalStoreCacheType finished:finished completion:^{
            // 存缓存数据和图片
            [self callStoreCacheProcessForOperation:operation url:url options:options context:context image:cacheImage data:cacheData cacheType:cacheType finished:finished completed:completedBlock];
        }];
    }
}

```
    
### 存储原始缓存过程主要代码：
1. 获取缓存类型SDImageCache
2. 默认存在磁盘缓存中
3. 根据SDWebImageContextCacheSerializer是否需要序列化、对图片按照指定的格式jpg/png进行缓存
    3.1 需要序列化，存储原始图片originalImage，序列化后的imageData；不需要序列化存储原始图片originalImage ，存储原始数据originalData
    storeImage:imageData:forKey:options:context:imageCache:cacheType:finished:completion:
    3.2 成功后存数缓存图片，序列化后的缓存图片
     callStoreCacheProcessForOperation:url:options:context:image:data:cacheType:finished:completed:

###    存储缓存过程
```
    // Store normal cache process
    - (void)callStoreCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                          url:(nonnull NSURL *)url
                                      options:(SDWebImageOptions)options
                                      context:(SDWebImageContext *)context
                                        image:(nullable UIImage *)image
                                         data:(nullable NSData *)data
                                    cacheType:(SDImageCacheType)cacheType
                                     finished:(BOOL)finished
                                    completed:(nullable SDInternalCompletionBlock)completedBlock {
        // Grab the image cache to use
        id<SDImageCache> imageCache = context[SDWebImageContextImageCache];
        if (!imageCache) {
            imageCache = self.imageCache;
        }
        // the target image store cache type
        SDImageCacheType storeCacheType = SDImageCacheTypeAll;
        if (context[SDWebImageContextStoreCacheType]) {
            storeCacheType = [context[SDWebImageContextStoreCacheType] integerValue];
        }
        id<SDWebImageCacheSerializer> cacheSerializer = context[SDWebImageContextCacheSerializer];
        
        // transformed cache key
        NSString *key = [self cacheKeyForURL:url context:context];
        if (finished && cacheSerializer && (storeCacheType == SDImageCacheTypeDisk || storeCacheType == SDImageCacheTypeAll)) {
            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                NSData *newData = [cacheSerializer cacheDataWithImage:image originalData:data imageURL:url];
                // Store image and data
                [self storeImage:image imageData:newData forKey:key options:options context:context imageCache:imageCache cacheType:storeCacheType finished:finished completion:^{
                    [self callCompletionBlockForOperation:operation completion:completedBlock image:image data:data error:nil cacheType:cacheType finished:finished queue:context[SDWebImageContextCallbackQueue] url:url];
                }];
            });
        } else {
            // Store image and data
            [self storeImage:image imageData:data forKey:key options:options context:context imageCache:imageCache cacheType:storeCacheType finished:finished completion:^{
                [self callCompletionBlockForOperation:operation completion:completedBlock image:image data:data error:nil cacheType:cacheType finished:finished queue:context[SDWebImageContextCallbackQueue] url:url];
            }];
        }
    }
```

这个过程跟存储原始缓存流程一致，唯一的差别是这个过程存的数据源来自缓存cacheImage、originalData，上一个过程存的数据来自原始数据originalImage、originalData
1. 获取缓存类型SDImageCache
2. 默认存在磁盘缓存中
3. 根据SDWebImageContextCacheSerializer是否需要序列化、对图片按照指定的格式jpg/png进行缓存
    3.1 需要序列化，存储原始图片originalImage，序列化后的imageData；不需要序列化存储原始图片originalImage ，存储原始数据originalData
    storeImage:imageData:forKey:options:context:imageCache:cacheType:finished:completion:
    3.2 成功后存数缓存图片，序列化后的缓存图片
     callStoreCacheProcessForOperation:url:options:context:image:data:cacheType:finished:completed:

###    存数图片、图片数据过程
storeImage:imageData:forKey:options:context:imageCache:cacheType:finished:completion:
```
    - (void)storeImage:(nullable UIImage *)image
             imageData:(nullable NSData *)data
                forKey:(nullable NSString *)key
               options:(SDWebImageOptions)options
               context:(nullable SDWebImageContext *)context
            imageCache:(nonnull id<SDImageCache>)imageCache
             cacheType:(SDImageCacheType)cacheType
              finished:(BOOL)finished
            completion:(nullable SDWebImageNoParamsBlock)completion {
        BOOL waitStoreCache = SD_OPTIONS_CONTAINS(options, SDWebImageWaitStoreCache);
        // 忽略渐进式数据缓存
        if (!finished) {
            if (completion) {
                completion();
            }
            return;
        }
        // 检查是否应等待存储缓存完成。如果不是,立即回调
        if ([imageCache respondsToSelector:@selector(storeImage:imageData:forKey:options:context:cacheType:completion:)]) {
            [imageCache storeImage:image imageData:data forKey:key options:options context:context cacheType:cacheType completion:^{
                if (waitStoreCache) {
                    if (completion) {
                        completion();
                    }
                }
            }];
        } else {
            [imageCache storeImage:image imageData:data forKey:key cacheType:cacheType completion:^{
                if (waitStoreCache) {
                    if (completion) {
                        completion();
                    }
                }
            }];
        }
        if (!waitStoreCache) {
            if (completion) {
                completion();
            }
        }
    }
```

这个过程主要：
1. 根据SDWebImageWaitStoreCache判断是否需要等待存数数据
2. 根据代理类型能否响应storeImage:imageData:forKey:options:context:cacheType:completion:、storeImage:imageData:forKey:context:cacheType:completion:执行对应的方法。这两个方法最终调用都是storeImage:imageData:forKey:options:context:cacheType:completion:
到这一步为止，一直是SDWebImageManager中执行的一系列代码，还未进行存储缓存，真正的缓存是在SDImageCache中实现。

# SDImageCache

##    存数图片、图片数据
storeImage:imageData:forKey:options:context:cacheType:completion:
```
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
           options:(SDWebImageOptions)options
           context:(nullable SDWebImageContext *)context
         cacheType:(SDImageCacheType)cacheType
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    // 纠错，直接返回
    if ((!image && !imageData) || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    BOOL toMemory = cacheType == SDImageCacheTypeMemory || cacheType == SDImageCacheTypeAll;
    BOOL toDisk = cacheType == SDImageCacheTypeDisk || cacheType == SDImageCacheTypeAll;
    // 默认存储到缓存中 NSCache或者NSMapTable
    if (image && toMemory && self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = image.sd_memoryCost;
        [self.memoryCache setObject:image forKey:key cost:cost];
    }
    
    // 不需要存到磁盘，直接返回完成
    if (!toDisk) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    
    // 存到磁盘中
    NSData *data = imageData;
    if (!data && [image respondsToSelector:@selector(animatedImageData)]) {
        // 如果图像是自定义动画图像类,优先选择其原始动画数据
        data = [((id<SDAnimatedImage>)image) animatedImageData];
    }
    SDCallbackQueue *queue = context[SDWebImageContextCallbackQueue];
    if (!data && image) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
            // 检查图像关联的图像格式,可能返回Undefined
            SDImageFormat format = image.sd_imageFormat;
            if (format == SDImageFormatUndefined) {
                // 如果图像是动画的,使用GIF(APNG可能更好,但在macOS 10.14之前有bug)
                if (image.sd_isAnimated) {
                    format = SDImageFormatGIF;
                } else {
                    // 如果我们没有任何数据来检测图像格式,请检查它是否包含alpha通道以使用PNG或JPEG格式
                    format = [SDImageCoderHelper CGImageContainsAlpha:image.CGImage] ? SDImageFormatPNG : SDImageFormatJPEG;
                }
            }
            // image转成序列化Data，存进磁盘需要data，打包对应的image信息
            NSData *data = [[SDImageCodersManager sharedManager] encodedDataWithImage:image format:format options:context[SDWebImageContextImageEncodeOptions]];
            dispatch_async(self.ioQueue, ^{
                [self _storeImageDataToDisk:data forKey:key];
                [self _archivedDataWithImage:image forKey:key];
                if (completionBlock) {
                    [(queue ?: SDCallbackQueue.mainQueue) async:^{
                        completionBlock();
                    }];
                }
            });
        });
    } else {
        dispatch_async(self.ioQueue, ^{
            [self _storeImageDataToDisk:data forKey:key];
            [self _archivedDataWithImage:image forKey:key];
            if (completionBlock) {
                [(queue ?: SDCallbackQueue.mainQueue) async:^{
                    completionBlock();
                }];
            }
        });
    }
}
```
###    存储主要做
1. 判断是否需要存储到缓存、磁盘中。默认都存缓存
    1.1 需要存储到缓存中memoryCache缓存图片（默认存储到缓存中 NSCache或者NSMapTable）
    1.2 不需要存储到磁盘，结束
    1.3 存储到磁盘中
        1.3.1 无动画数据animatedImageData，data = imageData，存储到diskCache中，并且把图片拓展信息extendedData打包后一并存入diskCache中
        1.3.2 有动画数据animatedImageData，data = animatedImageData，根据图片格式format用SDImageCodersManager编码成data，存储到diskCache中，并且把图片拓展信息extendedData打包后一并存入diskCache中

这样其中一条完整的流程已经结束，是从缓存中取出图片并且保存图片的流程。总结过程是，传入图片地址URL，进入入口函数后，查询缓存，并且保存缓存，如果缓存图片需要做改变，则保存改变后的图片和缓存，否则只保存缓存。

# 下载流程
-callDownloadProcessForOperation:url:options:context:cachedImage:cachedData:cacheType:progress:completed:

```
- (void)callDownloadProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                    url:(nonnull NSURL *)url
                                options:(SDWebImageOptions)options
                                context:(SDWebImageContext *)context
                            cachedImage:(nullable UIImage *)cachedImage
                             cachedData:(nullable NSData *)cachedData
                              cacheType:(SDImageCacheType)cacheType
                               progress:(nullable SDImageLoaderProgressBlock)progressBlock
                              completed:(nullable SDInternalCompletionBlock)completedBlock {
    // 进入下载流程，结束查缓存流程
    @synchronized (operation) {
        operation.cacheOperation = nil;
    }
    
    id<SDImageLoader> imageLoader = context[SDWebImageContextImageLoader];
    if (!imageLoader) {
        imageLoader = self.imageLoader;
    }
    
    // 是否需要下载
    // 非仅从缓存加载、需要强制刷新、响应shouldDownloadImageForURL、canRequestImageForURL、
    BOOL shouldDownload = !SD_OPTIONS_CONTAINS(options, SDWebImageFromCacheOnly);
    shouldDownload &= (!cachedImage || options & SDWebImageRefreshCached);
    shouldDownload &= (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
    if ([imageLoader respondsToSelector:@selector(canRequestImageForURL:options:context:)]) {
        shouldDownload &= [imageLoader canRequestImageForURL:url options:options context:context];
    } else {
        shouldDownload &= [imageLoader canRequestImageForURL:url];
    }
    
    // 下载的情况
    // 有缓存，但是需要强制刷新-重新下载
    
    if (shouldDownload) {
        if (cachedImage && options & SDWebImageRefreshCached) {
            // 如果在缓存中找到图像但提供了SDWebImageRefreshCached,则通知缓存中的图像, 并尝试重新下载它,以便让NSURLCache有机会从服务器重新刷新它。
            [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES queue:context[SDWebImageContextCallbackQueue] url:url];
            // 将缓存的图像传入图像加载器。图像加载器应检查远程图像是否等于缓存图像
            SDWebImageMutableContext *mutableContext;
            if (context) {
                mutableContext = [context mutableCopy];
            } else {
                mutableContext = [NSMutableDictionary dictionary];
            }
            mutableContext[SDWebImageContextLoaderCachedImage] = cachedImage;
            context = [mutableContext copy];
        }
        
        @weakify(operation);
        operation.loaderOperation = [imageLoader requestImageWithURL:url options:options context:context progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
            
            // 取消或者其他错误判断后走callCompletionBlockForOperation回调
            // 成功后走callTransformProcessForOperation进行所有参数回调
            // 需要transform，做transform
            // 根据context的缓存策略做SDImageCache的缓存
            // 如果需要做磁盘缓存，先做磁盘缓存（存在磁盘diskCache缓存中，同时归档拓展信息extendedData）
            // 没data数据，SDImageCodersManager解码
            // 做内存缓存imageCache
            // 将原始图片和transform后的图片都存到磁盘和缓存中
            // 最后走callCompletionBlockForOperation回调
            
            
            // 结束running序列中当前operation
            
            @strongify(operation);
            if (!operation || operation.isCancelled) { // 取消下载
                [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:@{NSLocalizedDescriptionKey : @"Operation cancelled by user during sending the request"}] queue:context[SDWebImageContextCallbackQueue] url:url];
            } else if (cachedImage && options & SDWebImageRefreshCached && [error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCacheNotModified) {
                // 图像刷新命中NSURLCache缓存,不调用completion块
            } else if ([error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCancelled) {
                // 在发送请求之前由用户取消的下载操作,不要封锁失败的URL
                [self callCompletionBlockForOperation:operation completion:completedBlock error:error queue:context[SDWebImageContextCallbackQueue] url:url];
            } else if (error) { // 下载失败，将URL放入失败池中
                [self callCompletionBlockForOperation:operation completion:completedBlock error:error queue:context[SDWebImageContextCallbackQueue] url:url];
                BOOL shouldBlockFailedURL = [self shouldBlockFailedURLWithURL:url error:error options:options context:context];
                
                if (shouldBlockFailedURL) {
                    SD_LOCK(self->_failedURLsLock);
                    [self.failedURLs addObject:url];
                    SD_UNLOCK(self->_failedURLsLock);
                }
            } else { // 下载成功，移除失败池中的URL -> 继续转场效果
                if ((options & SDWebImageRetryFailed)) {
                    SD_LOCK(self->_failedURLsLock);
                    [self.failedURLs removeObject:url];
                    SD_UNLOCK(self->_failedURLsLock);
                }
                [self callTransformProcessForOperation:operation url:url options:options context:context originalImage:downloadedImage originalData:downloadedData cacheType:SDImageCacheTypeNone finished:finished completed:completedBlock];
            }
            
            if (finished) {
                [self safelyRemoveOperationFromRunning:operation];
            }
        }];
    } else if (cachedImage) { // 取到缓存，直接用缓存数据
        [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES queue:context[SDWebImageContextCallbackQueue] url:url];
        [self safelyRemoveOperationFromRunning:operation];
    } else {// 即不成功，也不失败，其他情况下回调
        
        [self callCompletionBlockForOperation:operation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES queue:context[SDWebImageContextCallbackQueue] url:url];
        [self safelyRemoveOperationFromRunning:operation];
    }
}
```

##    下载流程主要代码：
1. 获取图片加载器
2. 判断是否需要下载（只取缓存SDWebImageFromCacheOnly、需要强制刷新SDWebImageRefreshCached、响应imageManager:shouldDownloadImageForURL:、imageManager:shouldDownloadImageForURL、imageLoader:canRequestImageForURL:options:context、imageLoader:canRequestImageForURL）
3. 不需要下载，判断是否有传入缓存cachedImage，有cachedImage，直接用缓存，执行结束回调，并且移除当前SDWebImageCombinedOperation
4. 不需要下载，也无传入缓存cachedImage，执行结束回调，并且移除当前SDWebImageCombinedOperation
4. 需要下载，
    4.1 先判断有缓存，执行回调刷新图片，继续执行
    4.2 loaderOperation，请求url-requestImageWithURL:options:context:progress:completed:
    4.3 加载url成功回调
        4.3.1 当前SDWebImageCombinedOperation已经取消，结束回调
        4.3.2 有缓存且需要强制刷新，且缓存不变，不做操作
        4.3.3 在加载结束前已经被取消，调用结束回调
        4.3.4 下载失败，将URL放入失败池中，调用结束回调
        4.3.5 下载成功，移除失败池中的URL -> 继续转场效果


请求URL-requestImageWithURL:options:context:progress:completed:
```
    - (id<SDWebImageOperation>)requestImageWithURL:(NSURL *)url options:(SDWebImageOptions)options context:(SDWebImageContext *)context progress:(SDImageLoaderProgressBlock)progressBlock completed:(SDImageLoaderCompletedBlock)completedBlock {
        UIImage *cachedImage = context[SDWebImageContextLoaderCachedImage];
        
        SDWebImageDownloaderOptions downloaderOptions = 0;
        if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
        if (options & SDWebImageProgressiveLoad) downloaderOptions |= SDWebImageDownloaderProgressiveLoad;
        if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
        if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
        if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
        if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
        if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
        if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
        if (options & SDWebImageAvoidDecodeImage) downloaderOptions |= SDWebImageDownloaderAvoidDecodeImage;
        if (options & SDWebImageDecodeFirstFrameOnly) downloaderOptions |= SDWebImageDownloaderDecodeFirstFrameOnly;
        if (options & SDWebImagePreloadAllFrames) downloaderOptions |= SDWebImageDownloaderPreloadAllFrames;
        if (options & SDWebImageMatchAnimatedImageClass) downloaderOptions |= SDWebImageDownloaderMatchAnimatedImageClass;
        
        if (cachedImage && options & SDWebImageRefreshCached) {
            // force progressive off if image already cached but forced refreshing
            downloaderOptions &= ~SDWebImageDownloaderProgressiveLoad;
            // ignore image read from NSURLCache if image if cached but force refreshing
            downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
        }
        
        return [self downloadImageWithURL:url options:downloaderOptions context:context progress:progressBlock completed:completedBlock];
    }

```

### 请求图片流程主要：
1. 将配置参数SDWebImageOptions转成下载配置参数SDWebImageDownloaderOptions
2. 执行下载流程-downloadImageWithURL:options:context:progress:completed:


执行下载流程-downloadImageWithURL:options:context:progress:completed:
```
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                   context:(nullable SDWebImageContext *)context
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    // URL将用作回调字典中的键,因此不能为nil。如果为nil,立即使用无图像或数据调用completed
    if (url == nil) {
        if (completedBlock) {
            NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
            completedBlock(nil, nil, error, YES);
        }
        return nil;
    }
    
    id downloadOperationCancelToken;
    // 当用相同的url下载不同大小的缩略图时,我们需要确保每个回调都以期望的大小调用
    id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
    NSString *cacheKey;
    if (cacheKeyFilter) {
        cacheKey = [cacheKeyFilter cacheKeyForURL:url];
    } else {
        cacheKey = url.absoluteString;
    }
    // 将downloadOption和context内置转换成SDImageCoderOption
    SDImageCoderOptions *decodeOptions = SDGetDecodeOptionsFromContext(context, [self.class imageOptionsFromDownloaderOptions:options], cacheKey); // 下载前，先获取解码参数（虽然之前尝试从缓存获取UIImage的时候已经计算过一次，不知道为什么这里又重新算一次，可能是代码虽然差不多，但是为了关键解耦，单独处理逻辑）
    SD_LOCK(_operationsLock);
    NSOperation<SDWebImageDownloaderOperation> *operation = [self.URLOperations objectForKey:url];
    // There is a case that the operation may be marked as finished or cancelled, but not been removed from `self.URLOperations`.
    BOOL shouldNotReuseOperation;
    if (operation) {
        @synchronized (operation) {
            shouldNotReuseOperation = operation.isFinished || operation.isCancelled || SDWebImageDownloaderOperationGetCompleted(operation);
        }
    } else {
        shouldNotReuseOperation = YES;
    }
    if (shouldNotReuseOperation) {
        // 读option和context创建并配置下载任务
        operation = [self createDownloaderOperationWithUrl:url options:options context:context];
        if (!operation) {
            SD_UNLOCK(_operationsLock);
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidDownloadOperation userInfo:@{NSLocalizedDescriptionKey : @"Downloader operation is nil"}];
                completedBlock(nil, nil, error, YES);
            }
            return nil;
        }
        @weakify(self);
        operation.completionBlock = ^{
            @strongify(self);
            if (!self) {
                return;
            }
            SD_LOCK(self->_operationsLock);
            [self.URLOperations removeObjectForKey:url];
            SD_UNLOCK(self->_operationsLock);
        };
        [self.URLOperations setObject:operation forKey:url];
        // Add the handlers before submitting to operation queue, avoid the race condition that operation finished before setting handlers.
        downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock decodeOptions:decodeOptions];
        // Add operation to operation queue only after all configuration done according to Apple's doc.
        // `addOperation:` does not synchronously execute the `operation.completionBlock` so this will not cause deadlock.
        [self.downloadQueue addOperation:operation];
    } else {
        // When we reuse the download operation to attach more callbacks, there may be thread safe issue because the getter of callbacks may in another queue (decoding queue or delegate queue)
        // So we lock the operation here, and in `SDWebImageDownloaderOperation`, we use `@synchonzied (self)`, to ensure the thread safe between these two classes.
        @synchronized (operation) {
            downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock decodeOptions:decodeOptions];
        }
    }
    SD_UNLOCK(_operationsLock);
    
    SDWebImageDownloadToken *token = [[SDWebImageDownloadToken alloc] initWithDownloadOperation:operation];
    token.url = url;
    token.request = operation.request;
    token.downloadOperationCancelToken = downloadOperationCancelToken;
    
    return token;
}
```
下载主要流程：
1. 生成下载操作SDImageCoderOptions
2. 判断是否需要下载操作
    2.1 需要下载操作
        2.1.1 根据配置生成SDWebImageDownloaderOperation下载操作
        2.1.2 添加进下载队列URLOperations中、downloadQueue中

    2.2 不需要下载操作

3. 返回最终下载SDWebImageDownloadToken

这样一个下载流程也补充完全。（下载完成后的流程这里不具体说明，详细来看看图片解码动作。）

### 下载完成后需要对图片数据进行解析
SDWebImageDownloaderOperationToken的成功-URLSession:task:didCompleteWithError:方法中，图像解码是一个串行队列，循环所有的下载SDWebImageDownloaderOperationToken
```
    // decode the image in coder queue, cancel all previous decoding process
    [self.coderQueue cancelAllOperations];
    @weakify(self);
    for (SDWebImageDownloaderOperationToken *token in tokens) {
        [self.coderQueue addOperationWithBlock:^{
            @strongify(self);
            if (!self) {
                return;
            }
            UIImage *image;
            // check if we already decode this variant of image for current callback
            if (token.decodeOptions) {
                image = [self.imageMap objectForKey:token.decodeOptions];
            }
            if (!image) {
                // check if we already use progressive decoding, use that to produce faster decoding
                id<SDProgressiveImageCoder> progressiveCoder = SDImageLoaderGetProgressiveCoder(self);
                SDWebImageOptions options = [[self class] imageOptionsFromDownloaderOptions:self.options];
                SDWebImageContext *context;
                if (token.decodeOptions) {
                    SDWebImageMutableContext *mutableContext = [NSMutableDictionary dictionaryWithDictionary:self.context];
                    SDSetDecodeOptionsToContext(mutableContext, &options, token.decodeOptions);
                    context = [mutableContext copy];
                } else {
                    context = self.context;
                }
                if (progressiveCoder) {
                    image = SDImageLoaderDecodeProgressiveImageData(imageData, self.request.URL, YES, self, options, context);
                } else {
                    image = SDImageLoaderDecodeImageData(imageData, self.request.URL, options, context);
                }
                if (image && token.decodeOptions) {
                    [self.imageMap setObject:image forKey:token.decodeOptions];
                }
            }
            CGSize imageSize = image.size;
            if (imageSize.width == 0 || imageSize.height == 0) {
                NSString *description = image == nil ? @"Downloaded image decode failed" : @"Downloaded image has 0 pixels";
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorBadImageData userInfo:@{NSLocalizedDescriptionKey : description}];
                [self callCompletionBlockWithToken:token image:nil imageData:nil error:error finished:YES];
            } else {
                [self callCompletionBlockWithToken:token image:image imageData:imageData error:nil finished:YES];
            }
        }];
    }
    // call [self done] after all completed block was dispatched
    dispatch_block_t doneBlock = ^{
        @strongify(self);
        if (!self) {
            return;
        }
        [self done];
    };
    if (@available(iOS 13, tvOS 13, macOS 10.15, watchOS 6, *)) {
        // seems faster than `addOperationWithBlock`
        [self.coderQueue addBarrierBlock:doneBlock];
    } else {
        // serial queue, this does the same effect in semantics
        [self.coderQueue addOperationWithBlock:doneBlock];
    }
```

图片会先从imageMap中取，如果没有缓存，则执行解码流程。解码流程分渐进式图像解码和普通解码。下面是渐进式的解码流程代码
```
UIImage * _Nullable SDImageLoaderDecodeProgressiveImageData(NSData * _Nonnull imageData, NSURL * _Nonnull imageURL, BOOL finished,  id<SDWebImageOperation> _Nonnull operation, SDWebImageOptions options, SDWebImageContext * _Nullable context) {
    NSCParameterAssert(imageData);
    NSCParameterAssert(imageURL);
    NSCParameterAssert(operation);
    
    UIImage *image;
    /// 缓存过滤器,来过滤和修改用于缓存的密钥
    id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
    NSString *cacheKey;
    if (cacheKeyFilter) {
        cacheKey = [cacheKeyFilter cacheKeyForURL:imageURL];
    } else {
        cacheKey = imageURL.absoluteString;
    }
    /// 获取解码配置
    SDImageCoderOptions *coderOptions = SDGetDecodeOptionsFromContext(context, options, cacheKey);
    // 解码第一帧
    BOOL decodeFirstFrame = SD_OPTIONS_CONTAINS(options, SDWebImageDecodeFirstFrameOnly);
    CGFloat scale = [coderOptions[SDImageCoderDecodeScaleFactor] doubleValue];
    
    // 逐步解码解码器
    id<SDProgressiveImageCoder> progressiveCoder = SDImageLoaderGetProgressiveCoder(operation);
    if (!progressiveCoder) {
        id<SDProgressiveImageCoder> imageCoder = context[SDWebImageContextImageCoder];
        // Check the progressive coder if provided
        if ([imageCoder respondsToSelector:@selector(initIncrementalWithOptions:)]) {
            progressiveCoder = [[[imageCoder class] alloc] initIncrementalWithOptions:coderOptions];
        } else {
            // We need to create a new instance for progressive decoding to avoid conflicts
            for (id<SDImageCoder> coder in [SDImageCodersManager sharedManager].coders.reverseObjectEnumerator) {
                if ([coder conformsToProtocol:@protocol(SDProgressiveImageCoder)] &&
                    [((id<SDProgressiveImageCoder>)coder) canIncrementalDecodeFromData:imageData]) {
                    // 生成对应的解码器类型
                    // 因为增量解码需要保留解码上下文,我们将为每个下载操作分配同一类的新实例,以避免冲突。
                    progressiveCoder = [[[coder class] alloc] initIncrementalWithOptions:coderOptions];
                    break;
                }
            }
        }
        SDImageLoaderSetProgressiveCoder(operation, progressiveCoder);
    }
    // If we can't find any progressive coder, disable progressive download
    if (!progressiveCoder) {
        return nil;
    }
    // 当有新的图像数据可用时更新增量解码
    [progressiveCoder updateIncrementalData:imageData finished:finished];
    if (!decodeFirstFrame) {
        // check whether we should use `SDAnimatedImage`
        Class animatedImageClass = context[SDWebImageContextAnimatedImageClass];
        if ([animatedImageClass isSubclassOfClass:[UIImage class]] && [animatedImageClass conformsToProtocol:@protocol(SDAnimatedImage)] && [progressiveCoder respondsToSelector:@selector(animatedImageFrameAtIndex:)]) {
            image = [[animatedImageClass alloc] initWithAnimatedCoder:(id<SDAnimatedImageCoder>)progressiveCoder scale:scale];
            if (image) {
                // Progressive decoding does not preload frames
            } else {
                // Check image class matching
                if (options & SDWebImageMatchAnimatedImageClass) {
                    return nil;
                }
            }
        }
    }
    if (!image) {
        // 增量解码当前图像数据以获取图像。
        image = [progressiveCoder incrementalDecodedImageWithOptions:coderOptions];
    }
    if (image) {
        BOOL shouldDecode = !SD_OPTIONS_CONTAINS(options, SDWebImageAvoidDecodeImage);
        BOOL lazyDecode = [coderOptions[SDImageCoderDecodeUseLazyDecoding] boolValue];
        if (lazyDecode) {
            // lazyDecode = NO means we should not forceDecode, highest priority
            shouldDecode = NO;
        }
        if (shouldDecode) {
            // 使用提供的图像返回解码后的图像。这与`CGImageCreateDecoded:不同,不会解码包含alpha通道或动画图像的图像。在iOS 15+上,这可能会使用UIImage.preparingForDisplay()`来使用CMPhoto进行解码,性能会比旧解决方案好。
            // 根据不同平台和系统版本进行不同的解码
            image = [SDImageCoderHelper decodedImageWithImage:image];
        }
        // assign the decode options, to let manager check whether to re-decode if needed
        image.sd_decodeOptions = coderOptions;
        // mark the image as progressive (completed one are not mark as progressive)
        image.sd_isIncremental = !finished;
    }
    
    return image;
}
```
下载+解码流程
1. 首先生成继承自NSOperation的SDWebImageDownloaderOperation，配置当前operation，并将operation添加到NSOperationQueue下载队列中，添加到下载队列会出发operation的start方法。
2. 如果operation的isCancelled为YES时，说明下载已经被取消，则结束下载。
3. 创建NSURLSessionTask并执行resume方法开始下载。
4. 当收到服务端的响应式根据code判断请求状态，如果是正常状态则发送正在接收response的通知以及下载进度。如果是304或者其他异常状态则cancel下载操作。
5. 在didReceiveData每次收到服务器返回的response时，给data数据中追加图片当前下载的data，并回调下载的进度。
6. 在didCompleteWithError下载结束时，如果下载成功则进行图片data解码，图片的缩放或者压缩操作，发送下载结束通知。若下载失败，则回调失败block。
    






























