---
layout: post
title:  "SDWebImage阅读笔记"
date:   2019-01-08
excerpt:  "本文是笔者阅读SDWebImage的笔记"
tag:
- SourceCode
comments: true
---

SDWebImage是一个异步图片加载库(下面笔者简称为SD)，下面笔者来分析下它的源代码(v4.4.3)。

> 本文中的代码，笔者删减了图片解码和编码等部分，具体可以参看源码。

## 接口

{% highlight objective_c %}
[cell.imageView sd_setImageWithURL:[NSURL URLWithString:@"http://example.com/image.jpg"]
                      placeholderImage:[UIImage imageNamed:@"placeholder"]];
{% endhighlight %}

这是我们平时用到最多的接口，而且基本上不需要我们再额外设置什么了，接口内部会帮我们做好。

上面的接口是 `UIImageView` `WebCache` 分类中的一个方法，除了上面看到的这种，还有很多可选的，将很多参数进行缺省，形成很多子方法，内部都去调用下面这个包含所有参数的方法。

`sd_setImageWithURL:placeholderImage:options:progress:completed:`

除了给 `UIImageView` 增加了 `WebCache` 分类，还给 `UIButton`, `MKAnnotationView` 增加了`WebCache` 分类，接口定义方式和 `UIImageView` 一致，下面笔者以 `UIImageView` 为例进行叙述。

之前说到因为 `WebCache` 分类添加到多个UI组件中，所以此时给基类 `UIView` 也增加了 `WebCache` 分类，子类中的 `sd_setImageWithURL` 方法都会去调用父类中提供的实现方法，方法如下:

`sd_internalSetImageWithURL:placeholderImage:options:operationKey:setImageBlock:progress:completed:context:`

同时给 `UIView` 额外添加了一个 `WebCacheOperation` 分类，该分类主要是为了提供取消当前控件的图片加载, 方法如下:

`- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key;`

`sd_internalSetImageWithURL` 具体实现笔者将在下载和缓存后进行叙述。

## 下载

{% highlight objective_c %}
@protocol SDWebImageOperation <NSObject>

- (void)cancel;

@end
{% endhighlight %}

说下载操作之前，先给出SD中定义的一个协议，包含了一个 cancel 方法，下载部分遵守了该协议，提供了取消的操作。

SD中定义了一个类 `SDWebImageDownloadToken`, 这个类就是用来表示一个下载任务，遵守了 `SDWebImageOperation` 协议，定义如下:

{% highlight objective_c %}
@interface SDWebImageDownloadToken : NSObject <SDWebImageOperation>

@property (nonatomic, strong, nullable) NSURL *url;
@property (nonatomic, strong, nullable) id downloadOperationCancelToken;

@end
{% endhighlight %}

`SDWebImageDownloader` 这个类就是提供给外界根据一个图片URL进行图片下载操作的类，下载方法如下所示，方法返回的就是一个 `SDWebImageDownloadToken` 对象。

`downloadImageWithURL:options:progress:completed:`

`SDWebImageDownloader` 内部真正去进行HTTP下载请求的则是 `SDWebImageDownloaderOperation` 类，这个类继承自 `NSOperation`, 同时遵守了一个下载操作的协议 `SDWebImageDownloaderOperationInterface`。

{% highlight objective_c %}
@protocol SDWebImageDownloaderOperationInterface <NSURLSessionTaskDelegate, NSURLSessionDataDelegate>
@required
- (nonnull instancetype)initWithRequest:(nullable NSURLRequest *)request
                              inSession:(nullable NSURLSession *)session
                                options:(SDWebImageDownloaderOptions)options;

- (nullable id)addHandlersForProgress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                            completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock;

- (BOOL)shouldDecompressImages;
- (void)setShouldDecompressImages:(BOOL)value;

- (nullable NSURLCredential *)credential;
- (void)setCredential:(nullable NSURLCredential *)value;

- (BOOL)cancel:(nullable id)token;

@optional
- (nullable NSURLSessionTask *)dataTask;

@end
{% endhighlight %}

这个协议包含的东西还是比较多的，同时也是为了可以使开发者自定义一个`NSOperation`子类并且遵守这个协议，这样就可以用自己的子类去替换`SDWebImageDownloaderOperation` 的操作。

下面笔者先给出 `SDWebImageDownloaderOperation` 内部的一些属性:

{% highlight objective_c %}
typedef NSMutableDictionary<NSString *, id> SDCallbacksDictionary;

@interface SDWebImageDownloaderOperation ()

@property (strong, nonatomic, nonnull) NSMutableArray<SDCallbacksDictionary *> *callbackBlocks;
@end
{% endhighlight %}

现在可以看到内部有个数组 `callbackBlocks` 用来存储多个下载任务，每个任务是一个字典。

{% highlight objective_c %}
- (nonnull instancetype)initWithRequest:(nullable NSURLRequest *)request
                              inSession:(nullable NSURLSession *)session
                                options:(SDWebImageDownloaderOptions)options {
    if ((self = [super init])) {
        _request = [request copy];
        _shouldDecompressImages = YES;
        _options = options;
        _callbackBlocks = [NSMutableArray new];
        _executing = NO;
        _finished = NO;
        _expectedSize = 0;
        _unownedSession = session;
        _callbacksLock = dispatch_semaphore_create(1);
    }
    return self;
}
{% endhighlight %}

上面方法是 `SDWebImageDownloaderOperation` 实现 `SDWebImageDownloaderOperationInterface` 协议中的初始化方法，也很简单，初始化了接下来下载操作的配置，包括 `callbackBlocks`。

{% highlight objective_c %}
static NSString *const kProgressCallbackKey = @"progress";
static NSString *const kCompletedCallbackKey = @"completed";
- (nullable id)addHandlersForProgress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                            completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    SDCallbacksDictionary *callbacks = [NSMutableDictionary new];
    if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
    if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
    [self.callbackBlocks addObject:callbacks];
    return callbacks;
}
{% endhighlight %}

上面方法依然是`SDWebImageDownloaderOperationInterface` 协议中的方法，我们可以看到实现，往`callbackBlocks`添加一个新的元素，而且我们也知道了`callbackBlocks`中存储的其实是每一次下载的 progressBlock 和 completedBlock。

{% highlight objective_c %}
- (BOOL)cancel:(nullable id)token {
    BOOL shouldCancel = NO;
    [self.callbackBlocks removeObjectIdenticalTo:token];
    if (self.callbackBlocks.count == 0) {
        shouldCancel = YES;
    }
    if (shouldCancel) {
        [self cancel];
    }
    return shouldCancel;
}
- (void)cancel {
    [self cancelInternal];
}
{% endhighlight %}

继续来看协议中的 `cancel:` 方法, 参数token就是上一步存储到`callbackBlocks`数组中的包含progressBlock 和 completedBlock的字典，判断数组为空，如果为空则cancel当前Operation。

{% highlight objective_c %}
- (void)start {
    if (self.isCancelled) {
        self.finished = YES;
        [self reset];
        return;
    }

    NSURLSession *session = self.unownedSession;
    if (!session) {
        NSURLSessionConfiguration *sessionConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
        sessionConfig.timeoutIntervalForRequest = 15;
            
        session = [NSURLSession sessionWithConfiguration:sessionConfig
                                                delegate:self
                                            delegateQueue:nil];
        self.ownedSession = session;
    }

    if (self.options & SDWebImageDownloaderIgnoreCachedResponse) {
            NSURLCache *URLCache = session.configuration.URLCache;
            if (!URLCache) {
                URLCache = [NSURLCache sharedURLCache];
            }
            NSCachedURLResponse *cachedResponse;
            // NSURLCache's `cachedResponseForRequest:` is not thread-safe, see https://developer.apple.com/documentation/foundation/nsurlcache#2317483
            @synchronized (URLCache) {
                cachedResponse = [URLCache cachedResponseForRequest:self.request];
            }
            if (cachedResponse) {
                self.cachedData = cachedResponse.data;
            }
    }
        
    self.dataTask = [session dataTaskWithRequest:self.request];
    self.executing = YES;

    if (self.dataTask) {
        [self.dataTask resume];
        for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            progressBlock(0, NSURLResponseUnknownLength, self.request.URL);
        }
    } else {
        [self callCompletionBlocksWithError:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorUnknown userInfo:@{NSLocalizedDescriptionKey : @"Task can't be initialized"}]];
        return;
    }
}
- (nullable NSArray<id> *)callbacksForKey:(NSString *)key {
    NSMutableArray<id> *callbacks = [[self.callbackBlocks valueForKey:key] mutableCopy];
    [callbacks removeObjectIdenticalTo:[NSNull null]];
    return [callbacks copy]; // strip mutability here
}
{% endhighlight %}

上面operation 的 start 方法就是实际发送网络请求的地方，当resume后，开始取`callbackBlocks`中 progressBlock进行第一次回调。同时如果需要URLCache将数据存储下来后面比对。

下面就是`SDWebImageDownloaderOperation`实现的 `NSURLSessionTaskDelegate` 和 `NSURLSessionDataDelegate` 的方法，进行网络请求的具体处理。

{% highlight objective_c %}
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
    NSURLSessionResponseDisposition disposition = NSURLSessionResponseAllow;
    NSInteger expected = (NSInteger)response.expectedContentLength;
    expected = expected > 0 ? expected : 0;
    self.expectedSize = expected;
    self.response = response;
   
    for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            progressBlock(0, expected, self.request.URL);
    }
    
    if (completionHandler) {
        completionHandler(disposition);
    }
}
{% endhighlight %}

当请求响应时，设置图片大小，progressBlock再次进行回调。

{% highlight objective_c %}
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
    if (!self.imageData) {
        self.imageData = [[NSMutableData alloc] initWithCapacity:self.expectedSize];
    }
    [self.imageData appendData:data];

    for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
        progressBlock(self.imageData.length, self.expectedSize, self.request.URL);
    }
}
{% endhighlight %}

这一步时根据返回的data进行更新progressBlock。

{% highlight objective_c %}
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    self.dataTask = nil;

    if (error) {
        [self callCompletionBlocksWithError:error];
        [self done];
    } else {
        if ([self callbacksForKey:kCompletedCallbackKey].count > 0) {
            __block NSData *imageData = [self.imageData copy];
            if (imageData) {
                /**  if you specified to only use cached data via `SDWebImageDownloaderIgnoreCachedResponse`,
                 *  then we should check if the cached data is equal to image data
                 */
                if (self.options & SDWebImageDownloaderIgnoreCachedResponse && [self.cachedData isEqualToData:imageData]) {
                    // call completion block with nil
                    [self callCompletionBlocksWithImage:nil imageData:nil error:nil finished:YES];
                    [self done];
                } else {
                    // decode the image in coder queue
                    dispatch_async(self.coderQueue, ^{
                        @autoreleasepool {
                            UIImage *image = [[SDWebImageCodersManager sharedInstance] decodedImageWithData:imageData];
                            NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
                            image = [self scaledImageForKey:key image:image];

                            if (self.shouldDecompressImages) {
                                BOOL shouldScaleDown = self.options & SDWebImageDownloaderScaleDownLargeImages;
                                image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&imageData options:@{SDWebImageCoderScaleDownLargeImagesKey: @(shouldScaleDown)}];
                            }

                            CGSize imageSize = image.size;
                            if (imageSize.width == 0 || imageSize.height == 0) {
                                [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Downloaded image has 0 pixels"}]];
                            } else {
                                [self callCompletionBlocksWithImage:image imageData:imageData error:nil finished:YES];
                            }
                            [self done];
                        }
                    });
                }
            } else {
                [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Image data is nil"}]];
                [self done];
            }
        } else {
            [self done];
        }
    }
}
- (void)callCompletionBlocksWithError:(nullable NSError *)error {
    [self callCompletionBlocksWithImage:nil imageData:nil error:error finished:YES];
}
- (void)callCompletionBlocksWithImage:(nullable UIImage *)image
                            imageData:(nullable NSData *)imageData
                                error:(nullable NSError *)error
                             finished:(BOOL)finished {
    NSArray<id> *completionBlocks = [self callbacksForKey:kCompletedCallbackKey];
    for (SDWebImageDownloaderCompletedBlock completedBlock in completionBlocks) {
        completedBlock(image, imageData, error, finished);
    }
}
{% endhighlight %}

请求结束，进行回调completedBlock:

- 判断`callbackBlocks`中是否存在completedBlock
- 判断之前接收的imageData是否为空
- 判断是否需要直接走URLCache 逻辑
- 解码imageData，尝试压缩图片
- 判断图片size
- 最后使用imageData和解码后的image，回调completedBlock

{% highlight objective_c %}
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler {
    
    NSCachedURLResponse *cachedResponse = proposedResponse;

    if (!(self.options & SDWebImageDownloaderUseNSURLCache)) {
        // Prevents caching of responses
        cachedResponse = nil;
    }
    if (completionHandler) {
        completionHandler(cachedResponse);
    }
}
{% endhighlight %}

最后是一个是否需要URLCache的回调，处理也很简单，如果设置了 `SDWebImageDownloaderUseNSURLCache` flag 则 cache response。

至此，SDWebImageDownloaderOperation的工作结束。

### SDWebImageDownloader

{% highlight objective_c %}
@interface SDWebImageDownloader () <NSURLSessionTaskDelegate, NSURLSessionDataDelegate>

@property (strong, nonatomic, nonnull) NSOperationQueue *downloadQueue;
@property (weak, nonatomic, nullable) NSOperation *lastAddedOperation;
@property (assign, nonatomic, nullable) Class operationClass;
@property (strong, nonatomic, nonnull) NSMutableDictionary<NSURL *, NSOperation<SDWebImageDownloaderOperationInterface> *> *URLOperations;
@property (strong, nonatomic, nullable) SDHTTPHeadersMutableDictionary *HTTPHeaders;
@property (strong, nonatomic, nonnull) dispatch_semaphore_t operationsLock; // a lock to keep the access to `URLOperations` thread-safe
@property (strong, nonatomic, nonnull) dispatch_semaphore_t headersLock; // a lock to keep the access to `HTTPHeaders` thread-safe

@property (strong, nonatomic) NSURLSession *session;

@end
{% endhighlight %}

{% highlight objective_c %}
typedef NSDictionary<NSString *, NSString *> SDHTTPHeadersDictionary;
typedef NSMutableDictionary<NSString *, NSString *> SDHTTPHeadersMutableDictionary;
- (nonnull instancetype)initWithSessionConfiguration:(nullable NSURLSessionConfiguration *)sessionConfiguration {
    if ((self = [super init])) {
        _operationClass = [SDWebImageDownloaderOperation class];
        _shouldDecompressImages = YES;
        _executionOrder = SDWebImageDownloaderFIFOExecutionOrder;
        _downloadQueue = [NSOperationQueue new];
        _downloadQueue.maxConcurrentOperationCount = 6;
        _downloadQueue.name = @"com.hackemist.SDWebImageDownloader";
        _URLOperations = [NSMutableDictionary new];
        SDHTTPHeadersMutableDictionary *headerDictionary = [SDHTTPHeadersMutableDictionary dictionary];

        NSString *userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];

        if (userAgent) {
            if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) {
                NSMutableString *mutableUserAgent = [userAgent mutableCopy];
                if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
                    userAgent = mutableUserAgent;
                }
            }
            headerDictionary[@"User-Agent"] = userAgent;
        }

        headerDictionary[@"Accept"] = @"image/*;q=0.8";

        _HTTPHeaders = headerDictionary;
        _operationsLock = dispatch_semaphore_create(1);
        _headersLock = dispatch_semaphore_create(1);
        _downloadTimeout = 15.0;

        [self createNewSessionWithConfiguration:sessionConfiguration];
    }
    return self;
}
{% endhighlight %}

现在回到 `SDWebImageDownloader`, 这个类内部有一个NSOperationQueue，同时初始化做了下载请求header的构造工作，同时有一个 `URLOperations` 字典存储URL&Operation。

{% highlight objective_c %}
- (NSOperation<SDWebImageDownloaderOperationInterface> *)createDownloaderOperationWithUrl:(nullable NSURL *)url
                                                                                  options:(SDWebImageDownloaderOptions)options {
    NSTimeInterval timeoutInterval = self.downloadTimeout;
    if (timeoutInterval == 0.0) {
        timeoutInterval = 15.0;
    }

    NSURLRequestCachePolicy cachePolicy = options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData;
    NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url
                                                                cachePolicy:cachePolicy
                                                            timeoutInterval:timeoutInterval];
    
    request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
    request.HTTPShouldUsePipelining = YES;
    request.allHTTPHeaderFields = [self allHTTPHeaderFields];

    NSOperation<SDWebImageDownloaderOperationInterface> *operation = [[self.operationClass alloc] initWithRequest:request inSession:self.session options:options];
    operation.shouldDecompressImages = self.shouldDecompressImages;
    
    if (self.urlCredential) {
        operation.credential = self.urlCredential;
    } else if (self.username && self.password) {
        operation.credential = [NSURLCredential credentialWithUser:self.username password:self.password persistence:NSURLCredentialPersistenceForSession];
    }
    
    if (options & SDWebImageDownloaderHighPriority) {
        operation.queuePriority = NSOperationQueuePriorityHigh;
    } else if (options & SDWebImageDownloaderLowPriority) {
        operation.queuePriority = NSOperationQueuePriorityLow;
    }

    return operation;
}
{% endhighlight %}

上面方法就是构造request创建一个DownloadOperation 的过程，默认也就是我们之前说的 `SDWebImageDownloaderOperation` 实例。


{% highlight objective_c %}
@interface SDWebImageDownloadToken ()

@property (nonatomic, weak, nullable) NSOperation<SDWebImageDownloaderOperationInterface> *downloadOperation;

@end
{% endhighlight %}

{% highlight objective_c %}
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return nil;
    }
    
    NSOperation<SDWebImageDownloaderOperationInterface> *operation = [self.URLOperations objectForKey:url];
    if (!operation || operation.isFinished) {
        operation = [self createDownloaderOperationWithUrl:url options:options];
        __weak typeof(self) wself = self;
        operation.completionBlock = ^{
            __strong typeof(wself) sself = wself;
            if (!sself) {
                return;
            }
            LOCK(sself.operationsLock);
            [sself.URLOperations removeObjectForKey:url];
            UNLOCK(sself.operationsLock);
        };
        [self.URLOperations setObject:operation forKey:url];
        [self.downloadQueue addOperation:operation];
    }

    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    
    SDWebImageDownloadToken *token = [SDWebImageDownloadToken new];
    token.downloadOperation = operation;
    token.url = url;
    token.downloadOperationCancelToken = downloadOperationCancelToken;

    return token;
}
{% endhighlight %}

前面说到 `SDWebImageDownloadToken`, 用来表示一个下载任务，extension中存储了一个 operation，同时现在我们知道 `downloadOperationCancelToken` 属性其实就是存储的progressBlock 和 completedBlock的字典。

之前说到 `SDWebImageDownloader` 类中提供给外界根据一个图片URL进行图片下载的方法实现其实很简单:

- 内部创建operation，将图片URL和operation存进内部的字典
- 将operation加入到队列中，operation进行下载
- 当operation执行结束，移除先前字典中存储的记录
- 对外界返回 SDWebImageDownloadToken 实例

SDWebImageDownloader 中也实现了 NSURLSessionTaskDelegate 和 NSURLSessionDataDelegate 的方法，不过没有处理，通过task找到对应的operation去处理。

{% highlight objective_c %}
- (void)cancel:(nullable SDWebImageDownloadToken *)token {
    NSURL *url = token.url;
    if (!url) {
        return;
    }
    NSOperation<SDWebImageDownloaderOperationInterface> *operation = [self.URLOperations objectForKey:url];
    if (operation) {
        BOOL canceled = [operation cancel:token.downloadOperationCancelToken];
        if (canceled) {
            [self.URLOperations removeObjectForKey:url];
        }
    }
}
{% endhighlight %}

SDWebImageDownloader 中也提供了给定一个 SDWebImageDownloadToken，尝试取消当前下载任务。内部会调用之前说过的Operation的cancel方法。

到这里，下载部分叙述完毕。

## 缓存

SDImageCache 包含了内存缓存和磁盘缓存，内存缓存使用了系统的NSCache，磁盘缓存则是直接的文件存储。

### 存

{% highlight objective_c %}
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    if (!image || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }

    if (self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = image.sd_memoryCost;
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            @autoreleasepool {
                NSData *data = imageData;
                // encode part
                [self _storeImageDataToDisk:data forKey:key];
            }
            
            if (completionBlock) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionBlock();
                });
            }
        });
    } else {
        if (completionBlock) {
            completionBlock();
        }
    }
}
- (void)_storeImageDataToDisk:(nullable NSData *)imageData forKey:(nullable NSString *)key {
    if (!imageData || !key) {
        return;
    }
    
    if (![self.fileManager fileExistsAtPath:_diskCachePath]) {
        [self.fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }
    
    NSString *cachePathForKey = [self defaultCachePathForKey:key];
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    
    [imageData writeToURL:fileURL options:self.config.diskCacheWritingOptions error:nil];
    
    if (self.config.shouldDisableiCloud) {
        [fileURL setResourceValue:@YES forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}
{% endhighlight %}

存储部分还是很直观的，内存存储直接使用 `setObject:forKey:cost`, 磁盘存储则将data写到指定文件中，这里的文件名用MD5处理过。

### 取

取的方法很多提供了很多中，不过笔者这里只介绍一种下面会说到的 `queryCacheOperationForKey:options:done:`。

{% highlight objective_c %}
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock {
    if (!key) {
        if (doneBlock) {
            doneBlock(nil, nil, SDImageCacheTypeNone);
        }
        return nil;
    }
    
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    NSOperation *operation = [NSOperation new];
    void(^queryDiskBlock)(void) =  ^{
        if (operation.isCancelled) {
            return;
        }
        
        @autoreleasepool {
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage;
            SDImageCacheType cacheType = SDImageCacheTypeNone;
            if (image) {
                diskImage = image;
                cacheType = SDImageCacheTypeMemory;
            } else if (diskData) {
                cacheType = SDImageCacheTypeDisk;
                diskImage = [self diskImageForKey:key data:diskData options:options];
                if (diskImage && self.config.shouldCacheImagesInMemory) {
                    NSUInteger cost = diskImage.sd_memoryCost;
                    [self.memCache setObject:diskImage forKey:key cost:cost];
                }
            }
            
            if (doneBlock) {
                if (options & SDImageCacheQueryDiskSync) {
                    doneBlock(diskImage, diskData, cacheType);
                } else {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        doneBlock(diskImage, diskData, cacheType);
                    });
                }
            }
        }
    };
    
    if (options & SDImageCacheQueryDiskSync) {
        queryDiskBlock();
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}
{% endhighlight %}

- 首先从内存中寻找，如果设置了只从内存中寻找，这个时候就结束了
- 从磁盘中寻找，如果内存中没有，则将imageData解码，同时加入到内存缓存中
- 最后回调 doneblock

缓存部分笔者这里不展开，只说了下面需要用到的几个方法。

## 接口实现

在说 `sd_internalSetImageWithURL:placeholderImage:options:operationKey:setImageBlock:progress:completed:context:` 方法实现之前，还需要介绍一个类 - SDWebImageManager。

SDWebImageManager 这个类主要是封装了之前说的下载和缓存类的功能，提供了方法给 UIView 的 webCache 分类使用。内部又定义了一个类，可以理解为下载和缓存的Item，因为内部会尝试去缓存中取，如果取不到，再去下载，方法返回的就是 SDWebImageCombinedOperation 。

{% highlight objective_c %}
@interface SDWebImageCombinedOperation : NSObject <SDWebImageOperation>

@property (assign, nonatomic, getter = isCancelled) BOOL cancelled;
@property (strong, nonatomic, nullable) SDWebImageDownloadToken *downloadToken;
@property (strong, nonatomic, nullable) NSOperation *cacheOperation;
@property (weak, nonatomic, nullable) SDWebImageManager *manager;

@end

@implementation SDWebImageCombinedOperation

- (void)cancel {
    @synchronized(self) {
        self.cancelled = YES;
        if (self.cacheOperation) {
            [self.cacheOperation cancel];
            self.cacheOperation = nil;
        }
        if (self.downloadToken) {
            [self.manager.imageDownloader cancel:self.downloadToken];
        }
        [self.manager safelyRemoveOperationFromRunning:self];
    }
}

@end
{% endhighlight %}

SDWebImageCombinedOperation 实现了 SDWebImageOperation 协议，提供了cancel 方法。内部调用了下载的cancel方法。

`loadImageWithURL:options:progress:completed:`是SD中最长的一个方法，不过最新的5.x版本，已经将这个方法进行了拆分,下面笔者将该方法进行拆分来说明:

{% highlight objective_c %}
@interface SDWebImageManager ()

@property (strong, nonatomic, readwrite, nonnull) SDImageCache *imageCache;
@property (strong, nonatomic, readwrite, nonnull) SDWebImageDownloader *imageDownloader;
@property (strong, nonatomic, nonnull) NSMutableSet<NSURL *> *failedURLs;
@property (strong, nonatomic, nonnull) dispatch_semaphore_t failedURLsLock; // a lock to keep the access to `failedURLs` thread-safe
@property (strong, nonatomic, nonnull) NSMutableSet<SDWebImageCombinedOperation *> *runningOperations;
@property (strong, nonatomic, nonnull) dispatch_semaphore_t runningOperationsLock; // a lock to keep the access to `runningOperations` thread-safe

@end
{% endhighlight %}

> 首先给出SDWebImageManager中的属性。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }
}
{% endhighlight %}

这一步对传进来的url进行合法性判断，防止crash的产生。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    operation.manager = self;

    BOOL isFailedUrl = NO;
    if (url) {
        LOCK(self.failedURLsLock);
        isFailedUrl = [self.failedURLs containsObject:url];
        UNLOCK(self.failedURLsLock);
    }

    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil] url:url];
        return operation;
    }
}
{% endhighlight %}

failedURLs 是内部一个 mutableSet 用来存储无效的图片URL，这一步主要是判断如果图片URL无效，则直接返回，不尝试下载了。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    LOCK(self.runningOperationsLock);
    [self.runningOperations addObject:operation];
    UNLOCK(self.runningOperationsLock);
    NSString *key = [self cacheKeyForURL:url];
    
    SDImageCacheOptions cacheOptions = 0;
    if (options & SDWebImageQueryDataWhenInMemory) cacheOptions |= SDImageCacheQueryDataWhenInMemory;
    if (options & SDWebImageQueryDiskSync) cacheOptions |= SDImageCacheQueryDiskSync;
    if (options & SDWebImageScaleDownLargeImages) cacheOptions |= SDImageCacheScaleDownLargeImages;
}
{% endhighlight %}

runningOperations 是内部存储的当前正在执行的operation，这一步主要根据 WebImageManager 中 options 准备 cache 的 options。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    __weak SDWebImageCombinedOperation *weakOperation = operation;

    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
         __strong __typeof(weakOperation) strongOperation = weakOperation;
    }];

    return operation;
}
{% endhighlight %}

使用内部 imageCache 实例调用 queryCache 方法，同时将方法返回的一个NSOperation赋值给 内部的combinedOperation。主要的逻辑都在doneBlock中。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
        if (!strongOperation || strongOperation.isCancelled) {
            [self safelyRemoveOperationFromRunning:strongOperation];
            return;
        }
    }];

    return operation;
}
- (void)safelyRemoveOperationFromRunning:(nullable SDWebImageCombinedOperation*)operation {
    if (!operation) {
        return;
    }
    LOCK(self.runningOperationsLock);
    [self.runningOperations removeObject:operation];
    UNLOCK(self.runningOperationsLock);
}
{% endhighlight %}

这一步判断combinedOperation 是否被cancel，否则直接从runningOperations中删除这个operation。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
        BOOL shouldDownload = (!(options & SDWebImageFromCacheOnly))
            && (!cachedImage || options & SDWebImageRefreshCached)
            && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
    }];

    return operation;
}
{% endhighlight %}

这一步根据外界的设置，判断是否需要去下载图片。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
       if (shouldDownload) {
            if (cachedImage && options & SDWebImageRefreshCached) {
                [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            }
       }
    }];

    return operation;
}
{% endhighlight %}

这一步判断如果存在cachedImage 同时 设置了 SDWebImageRefreshCached flag，则首先回调 completedBlock，然后重新下载。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
       if (shouldDownload) {
            SDWebImageDownloaderOptions downloaderOptions = 0;
            if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
            if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
            if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
            if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
            if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
            if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
            if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
            if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
            
            if (cachedImage && options & SDWebImageRefreshCached) {
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
       }
    }];

    return operation;
}
{% endhighlight %}

这一步准备下载options。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
       if (shouldDownload) {
            __weak typeof(strongOperation) weakSubOperation = strongOperation;
            strongOperation.downloadToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
            }];
       }
    }];

    return operation;
}
{% endhighlight %}

这里则使用内部 imageDownloader 的 下载方法进行下载图片了，同时将返回的downToken赋值给combinedOperation。主要逻辑在下载方法的completedBlock中。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
       if (shouldDownload) {
            __weak typeof(strongOperation) weakSubOperation = strongOperation;
            strongOperation.downloadToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
                 __strong typeof(weakSubOperation) strongSubOperation = weakSubOperation;
                if (!strongSubOperation || strongSubOperation.isCancelled) { 
                    // 当operation被cancel，什么也不做。
                } else if (error) {
                    [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock error:error url:url];
                    BOOL shouldBlockFailedURL;
                    if ([self.delegate respondsToSelector:@selector(imageManager:shouldBlockFailedURL:withError:)]) {
                        shouldBlockFailedURL = [self.delegate imageManager:self shouldBlockFailedURL:url withError:error];
                    } else {
                        shouldBlockFailedURL = (   error.code != NSURLErrorNotConnectedToInternet
                                                && error.code != NSURLErrorCancelled
                                                && error.code != NSURLErrorTimedOut
                                                && error.code != NSURLErrorInternationalRoamingOff
                                                && error.code != NSURLErrorDataNotAllowed
                                                && error.code != NSURLErrorCannotFindHost
                                                && error.code != NSURLErrorCannotConnectToHost
                                                && error.code != NSURLErrorNetworkConnectionLost);
                    }
                    
                    if (shouldBlockFailedURL) {
                        LOCK(self.failedURLsLock);
                        [self.failedURLs addObject:url];
                        UNLOCK(self.failedURLsLock);
                    }
                } else {
                     if ((options & SDWebImageRetryFailed)) {
                        LOCK(self.failedURLsLock);
                        [self.failedURLs removeObject:url];
                        UNLOCK(self.failedURLsLock);
                    }
                }
            }];
       }
    }];

    return operation;
}
{% endhighlight %}

当下载完成，如果下载失败，根据设置将无效的url添加到failedURLsLock中。同时当下载成功根据设置，尝试从failedURLsLock移除之前添加的无效url。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
       if (shouldDownload) {
            __weak typeof(strongOperation) weakSubOperation = strongOperation;
            strongOperation.downloadToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
                 __strong typeof(weakSubOperation) strongSubOperation = weakSubOperation;
                if (!strongSubOperation || strongSubOperation.isCancelled) { 
                } else if (error) {
                } else {
                    BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);

                     if (options & SDWebImageRefreshCached && cachedImage && !downloadedImage) {
                        // Image refresh hit the NSURLCache cache, do not call the completion block
                    } else {
                        if (downloadedImage && finished) {
                           [self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key toDisk:cacheOnDisk completion:nil];
                        }
                        [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                    }
                }

                 if (finished) {
                    [self safelyRemoveOperationFromRunning:strongSubOperation];
                }
            }];
       }
    }];

    return operation;
}
{% endhighlight %}

- 当图片从URLCache中获得，不回调completedBlock
- 去除了关于图片transform的部分
- 这里则是图片成功下载，将其缓存，同时回调completedBlock

最后如果下载完成，runningOperations中删除当前operation。

{% highlight objective_c %}
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
       if (shouldDownload) {
       } else if (cachedImage) {
            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            [self safelyRemoveOperationFromRunning:strongOperation];
       } else {
            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
            [self safelyRemoveOperationFromRunning:strongOperation];
       }
    }];

    return operation;
}
{% endhighlight %}

- 有缓存的时候不让下载，这里判断cachedImage是否存在，回调completedBlock，runningOperations中删除当前operation。
- 缓存不存在，代理方法不让下载的情况。

至此，分析完了 SDWebImageManager 加载图片的实现，我们可以发现这个过程虽然比较多，不过都是建立在之前我们说过的缓存和下载两部分。

现在我们可以看最开始说到的 `sd_internalSetImageWithURL:placeholderImage:options:operationKey:setImageBlock:progress:completed:context:` 方法了。

{% highlight objective_c %}
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context {

    NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock];
        });
    }
}
{% endhighlight %}

这里首先取消当前View上的加载操作。然后设置占位图。

{% highlight objective_c %}
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context {
    if (url) {
        self.sd_imageProgress.totalUnitCount = 0;
        self.sd_imageProgress.completedUnitCount = 0;
        
        SDWebImageManager *manager = [context objectForKey:SDWebImageExternalCustomManagerKey];
        if (!manager) {
            manager = [SDWebImageManager sharedManager];
        }
        
        __weak __typeof(self)wself = self;
        SDWebImageDownloaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            wself.sd_imageProgress.totalUnitCount = expectedSize;
            wself.sd_imageProgress.completedUnitCount = receivedSize;
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
    } else {
        if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                completedBlock(nil, error, SDImageCacheTypeNone, url);
        }
    }
}
{% endhighlight %}

- url不为空，准备下载进度回调，初始化 SDWebImageManager。
- url为空，直接回调completedBlock

{% highlight objective_c %}
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context {
    if (url) {
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
        }];
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } 
}
{% endhighlight %}

这一步就是使用 SDWebImageManager 的获取图片方法，完成后将返回的 combineOperation&URL 存储到分类的字典中。

{% highlight objective_c %}
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context {
    if (url) {
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!sself) { return; }
                if (!shouldNotSetImage) {
                    [sself sd_setNeedsLayout];
                }
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, error, cacheType, url);
                }
            };

            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }

            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                targetImage = placeholder;
                targetData = nil;
            }

            SDWebImageTransition *transition = nil;
            [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];

            callCompletedBlockClojure();
        }];
    } 
}
{% endhighlight %}

- shouldCallCompletedBlock 是否回调 completedBlock， shouldNotSetImage 是否设置图片，否则默认用户自己在 completedBlock中取image设置
- 定义了一个完成的回调 callCompletedBlockClojure
- 如果 shouldNotSetImage 为true，执行 callCompletedBlockClojure，返回
- shouldNotSetImage 为false，设置图片到View，执行 callCompletedBlockClojure

至此，图片就可以显示到对应的View上。

## 最后

看完了SD，对网络图片异步加载的过程有了新的认识，其中关于多线程和图片编解码的内容还需要后面进行补充。

写完感觉文章比较冗长，还需要多多改进。
