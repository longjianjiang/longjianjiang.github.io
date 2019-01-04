---
layout: post
title:  "iOS Notification笔记"
date:   2018-08-26
excerpt:  "本文是笔者对iOS通知相关内容的一个记录。"
tag:
- iOS
comments: true
---

本文是笔者对iOS通知相关内容的一个记录。

iOS通知分为本地通知和远程通知，本文主要是远程通知，如是本地通知会具体说明。

其实处理通知相关，主要知道AppDelegate里关于通知的方法，和UserNotifications相关的方法即可。

iOS10开始，苹果重新封装了一个UserNotifications的framework，所以目前我们为了适配iOS9, 通知处理分为两种。

# 注册通知

1> 也就是系统会弹出一个alert让用户允许是否开启通知

## iOS10 之前：

{% highlight objective_c %}
UIUserNotificationType notificationTypes = UIUserNotificationTypeBadge | UIUserNotificationTypeSound | UIUserNotificationTypeAlert;
UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:notificationTypes categories:nil];
[application registerUserNotificationSettings:settings];
{% endhighlight %}

注意如果App没有注册远程通知，也就是不进行下面的第二步，此时想要获取用户是否允许通知，需要实现👇方法

{% highlight objective_c %}
- (void)application:(UIApplication *)application didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings {
  if (notificationSettings.types == UIUserNotificationTypeNone) {
    NSLog(@"user deny notification");
  } else {
    NSLog(@"user allow notification");
  }
}
{% endhighlight %}


## iOS10 之后：

{% highlight objective_c %}
UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
        [center requestAuthorizationWithOptions:UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound completionHandler:^(BOOL granted, NSError * _Nullable error) {
            if (granted) {
           } else {
                NSLog(@"User denied notification.");
            }
        }];
{% endhighlight %}

2> 注册远程通知
上一步只是开启了App的通知权限，不过因为我们需要远程推送，还需要调用下面方法来注册APNs服务：

{% highlight objective_c %}
[application registerForRemoteNotifications];
{% endhighlight %}

调用该方法会回调👇两个方法中其中一个：


{% highlight objective_c %}
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{% endhighlight %}

注册远程通知成功，获得了deviceToken，将其上报给第三方SDK。

{% highlight objective_c %}
- (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error
{% endhighlight %}

注册远程通知失败，一般模拟器上会失败。

此时我们已经做好了通知的准备工作，下面就等着接收通知。

# 监听通知

1> 远程通知

## iOS10 之前：

{% highlight objective_c %}
- (void)application:(UIApplication*)application didReceiveRemoteNotification:(NSDictionary*)userInfo
{% endhighlight %}

当用户收到一个远程通知会走该方法

{% highlight objective_c %}
- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification 
{% endhighlight %}

当用户收到一个本地通知会走该方法


## iOS10 之后：

新的UserNotifications框架统一了本地通知和远程通知的入口，两个回调方法都在 `UNUserNotificationCenterDelegate` 中。

{% highlight objective_c %}
-(void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler
{% endhighlight %}

当用户在前台收到一个通知，我们可以自定义是否要显示该通知，以及以什么样的方式显示，当不执行completionHandler则不会显示通知，执行completionHandler根据传入的UNNotificationPresentationOptions，以什么方式进行通知。

{% highlight objective_c %}
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)(void))completionHandler
{% endhighlight %}

该方法当用户点击通知，是我们处理用户对一条通知操作的地方。

2> 静默通知

{% highlight objective_c %}
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler 
{% endhighlight %}

静默通知是iOS7开始新加的一种类型的通知，属于特殊的远程推送通知，其目的不是为了弹出通知框提醒用户，而是用于后台运行的App和服务端同步数据。

推送数据格式, aps 字典中需添加 ` "content-available" : 1 ` ，用来标记是静默通知。
有了这个标记后不论App在前台还是后天，只要有通知，都会走到该方法，当App被杀死则不会走到了。


# 通知Category

iOS 10 之后我们可以注册通知Category，这样可以根据用户对通知的不同action进行处理，注册通知代码如下：

{% highlight objective_c %}
- (void)createCustomNotificationCategory {
  // 自定义`action1`和`action2`
  UNNotificationAction *action1 = [UNNotificationAction actionWithIdentifier:@"action1" title:@"test1" options: UNNotificationActionOptionNone];
  UNNotificationAction *action2 = [UNNotificationAction actionWithIdentifier:@"action2" title:@"test2" options: UNNotificationActionOptionNone];
  
  // 创建id为`test_category`的category，并注册两个action到category
  // UNNotificationCategoryOptionCustomDismissAction表明可以触发通知的dismiss回调
  UNNotificationCategory *category = [UNNotificationCategory categoryWithIdentifier:@"test_category" actions:@[] intentIdentifiers:@[] options:
                                      UNNotificationCategoryOptionCustomDismissAction];
  // 注册category到通知中心
  [[UNUserNotificationCenter currentNotificationCenter] setNotificationCategories:[NSSet setWithObjects:category, nil]];
}
{% endhighlight %}

注册完以后当用户点击通知就可以根据不同的action进行处理，处理代码如下：

{% highlight objective_c %}
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)(void))completionHandler {
    NSString *userAction = response.actionIdentifier;
    // ① 点击通知打开
    if ([userAction isEqualToString:UNNotificationDefaultActionIdentifier]) {
        JXLogDebug(@"User opened the notification.");
    }
  
    // ② 通知dismiss，category创建时传入UNNotificationCategoryOptionCustomDismissAction才可以触发
    if ([userAction isEqualToString:UNNotificationDismissActionIdentifier]) {
        JXLogDebug(@"User dismissed the notification.");
    }
  
    // ③ Action事件处理
    NSString *customAction1 = @"action1";
    NSString *customAction2 = @"action2";
    // 点击用户自定义Action1
    if ([userAction isEqualToString:customAction1]) {
        JXLogDebug(@"User custom action1.");
    }
    // 点击用户自定义Action2
    if ([userAction isEqualToString:customAction2]) {
        JXLogDebug(@"User custom action2.");
    }
  
    completionHandler();
}
{% endhighlight %}


# 通知Service Extension

这个Extension会在通知发送之前对通知进行一次操作，不论App有没有被杀死，Extension中的方法都会走到。
在这个Extension中我们可以改变通知内容，一般是传递一些敏感数据时内容是加密的，当到达App是客户端用对应的key进行解密，这样保证了即使传输过程中内容被截取了，不过内容是加密的，这样保证了数据的安全性。

而笔者使用这个Extension倒不是这个目的，而是为了保存通知到本地。因为笔者公司的一个App，点评列表服务端没有给列表接口，而是是客户端存储通知，列表从本地读取，所以笔者使用了Service Extension。因为当App被杀死的时候，用户收到通知，当用户删除该通知的时候，此时点评列表就等于少了一条记录，所以需要有一种方法不论怎样，每次通知到来都能执行到，而Service Extension 恰好满足需求。

注意此时我们需要在aps字典 中添加 `"mutable-content" : 1`

## App Group

因为要存储通知消息到本地，因为Extension与App是独立的，所以需要建立一个App Group来将存储内容放到一个公共的区域，这样Extension中存储的地方，和App读取的地方是同一个，下面是系统提供的一个公共区域用来同步数据：

{% highlight objective_c %}
NSString *appGroupName = @"";
NSURL *containerURL = [[NSFileManager defaultManager] containerURLForSecurityApplicationGroupIdentifier:appGroupName];
{% endhighlight %}


# 最后

未完待续




