
# AVPlayer 播放视频

## 基本流程：

- 使用url创建AVURLAsset；

- 使用<AVAsynchronousKeyValueLoading>协议中`loadValuesAsynchronouslyForKeys`方法进行prepare；

> 刚创建好的asset，还没有加载tracks，duration等信息，所以需要进行播放之前，需要去主动去获取这些信息，获取成功以后才可以进行播放。

- prepare 回调的时候，使用asset创建一个`AVPlayerItem`，这样`AVPlayer`拿到playerItem就可以进行播放视频，控制视频了。

## 相关类结构

- AVAsset

这个类代表了多媒体资源，有AVURLAsset、AVComposition两个子类，前者用来读取，后者用来编辑；

AVAssetTrack 表示多媒体资源中的轨道，一般视频有两个track，一个播放画面一个播放声音。

- AVPlayerItem

根据上面的播放流程，可以发现AVPlayer其实最终是根据AVPlayerItem中的内容来进行播放的。AVPlayerItem是对AVAsset的一层抽象，我们通过它来管理asset的呈现状态。

与之对应的，AVPlayerItemTrack是对AVAssetTrack的抽象，AVPlayerItem包含了若干AVPlayerItemTrack。

通过这层抽象可以做到用不同的渲染模式进行播放同一段多媒体。比如可以禁用某个声音的track，达到静音播放。

- AVPlayer

实际用来播放的类，可以进行播放控制，play，pause，seek等操作。同时可以监听播放时间，缓冲进度。

但是AVPlayer本身不能显示画面。

- AVPlayerLayer

这个类是用来展示AVPlayer中的画面。

---

结构如下图所示:

![avfoundation_note_1]({{site.url}}/assets/images/blog/avfoundation_note_1.png)

# AVPlayer 边播放边缓存

- 本地搭建http服务器；

- AVPlayer播放完毕，使用AVAsset将音频和视频的track添加到Composition，导出到沙盒；

- 使用AVAssetResourceLoader 回调手动接管整个数据的下载过程；

# References

[Apple Document](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html)

[AVFoundation Resources](https://developer.apple.com/av-foundation/)

[https://github.com/vitoziv/VIMediaCache/tree/master](https://github.com/vitoziv/VIMediaCache/tree/master)

[https://sky-weihao.github.io/2015/10/06/Video-streaming-and-caching-in-iOS/](https://sky-weihao.github.io/2015/10/06/Video-streaming-and-caching-in-iOS/)
