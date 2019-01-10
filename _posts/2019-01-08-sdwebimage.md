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

> 本文中的代码，笔者删减了部分，具体可以参看源码。

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
                            
                            BOOL shouldDecode = YES;

                            if (shouldDecode) {
                                if (self.shouldDecompressImages) {
                                    BOOL shouldScaleDown = self.options & SDWebImageDownloaderScaleDownLargeImages;
                                    image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&imageData options:@{SDWebImageCoderScaleDownLargeImagesKey: @(shouldScaleDown)}];
                                }
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
{% endhighlight %}

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

最后是一个是否需要URLCache的回调，处理也很简单，如果设置了SDWebImageDownloaderUseNSURLCache flag 则cache 响应。

至此，SDWebImageDownloaderOperation的工作结束。

## 缓存

## 接口实现
