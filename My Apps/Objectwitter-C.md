## 什么是 Objectwitter-C

Objectwitter-C 是一款稳定，成熟，全面的 Twitter REST/Streaming API 的 Objective-C 封装，其封装了 Twitter 的全部公开 API。由 [@NSTongG](https://twitter.com/NSTongG) 基于 [STTwitter](https://github.com/nst/STTwitter) 开发，在 STTwitter 对 [Twitter REST API](https://dev.twitter.com/rest/public) 的全面封装的基础上，增添了很多工具类使得 API 更加抽象，易用，并且完全重新设计了 [Twitter Streaming API](https://dev.twitter.com/streaming/overview) 的封装，利用类似于 [NSURLSession](https://developer.apple.com/library/mac/documentation/Foundation/Reference/NSURLSession_class/index.html)/[NSURLConnection](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSURLConnection_Class/) 的委托模式（delegate）使得用户更容易与 Twitter 流（streaming）API 交互。

Objectwitter-C 不依赖于 AppKit 和 UIKit 即可工作，你可以将该框架用在命令行应用中。你可以把 Objectwitter-C 考虑为 FOSS 版的 [Twitter Fabric](https://get.fabric.io) TwitterKit，但是去掉了 UI 部分并且更加灵活。

## 安装

将项目目录拖动到你自己的工程中，将你的工程与下列系统框架连接：

* Accounts.framework
* Social.framework
* Twitter.framework (iOS only)
* Security.framework (OS X only)

## 代码片段

初始化 Twitter API 对象：

```obj-c
STTwitterAPI* twitter = [ STTwitterAPI twitterAPIWithOAuthConsumerKey: @""
                                                       consumerSecret: @""
                                                             username: @""
                                                             password: @"" ];
```

验证凭据：

```obj-c
[ twitter verifyCredentialsWithUserSuccessBlock:
    ^( NSString* username, NSString* userID ) 
        {
        // ...
        } errorBlock:
            ^( NSError* error ) 
                {
                // ...
                } ];
```

抓取时间轴（Timeline）中的推文（tweets）：

```obj-c
[ twitter getHomeTimelineSinceID: nil
                           count: 100
                    successBlock:
    ^( NSArray* statuses ) 
        {
        // ...
        } errorBlock:
            ^( NSError* error ) 
                {
                // ...
                } ];
```

App Only 验证：

```obj-c
STTwitterAPI* twitter = [ STTwitterAPI twitterAPIAppOnlyWithConsumerKey: @""
                                                         consumerSecret: @"" ];
[ twitter verifyCredentialsWithUserSuccessBlock:
    ^( NSString* username, NSString* userID ) 
        {
        [ twitter getUserTimelineWithScreenName: @"barackobama"
                                   successBlock: 
            ^( NSArray* statuses ) 
                {
                // ...
                } errorBlock:
                    ^( NSError* error ) 
                        {
                        // ...
                        } ];
        } errorBlock: 
            ^( NSError* error ) 
                {
                // ...
                } ];
```

使用游标（cursor）枚举返回结果：

```obj-c
[ _twitter fetchAndFollowCursorsForResource: @"followers/ids.json"
                                 HTTPMethod: @"GET"
                              baseURLString: @"https://api.twitter.com/1.1"
                                 parameters: @{@"screen_name":@"0xcharlie"}
                        uploadProgressBlock: nil
                      downloadProgressBlock: nil
                               successBlock:
    ^( id request, NSDictionary* requestHeaders, NSDictionary* responseHeaders, id response, BOOL morePagesToCome, BOOL* stop ) 
        {
        NSLog( @"-- success, more to come: %d, %@", morePagesToCome, response );
        }                        pauseBlock:
    ^( NSDate* nextRequestDate ) 
        {
        NSLog( @"-- rate limit exhausted, nextRequestDate: %@", nextRequestDate );
        }                        errorBlock: 
    ^( id request, NSDictionary* requestHeaders, NSDictionary* responseHeaders, NSError* error ) 
        {
        NSLog( @"-- %@", error );
        } ];
```

不同类型的 OAuth 连接：你可以以三种方法初始化 STTwitterAPI 对象：

* 使用 OS X Preferences 或者 iOS Settings 中的 Twitter 账号
* 使用你自己指定的 consumer key 和 consumer secret（四种风格）：
    * 获取一个 URL，抓取 PIN 码输入到你的 app 中从而获取 OAuth access token
    * 设置用户名（username）和密码（password），然后使用 XAuth 获取 OAuth access token
    * 直接设置 OAuth token 和 OAuth token secret
    * 打开 Safari 或者使用 UIWebView 实例，执行 Twitter 验证操作然后通过一个定制的 UEL scheme 在你的 app 中直接接收 OAuth access token
* 使用 [Application Only](https://dev.twitter.com/oauth/application-only) 验证，获取并使用 “bearer token“

有五个 API 对应五种情况：

```obj-c
+ ( STTwitterAPI* ) twitterAPIOSWithFirstAccount;

+ ( STTwitterAPI* ) twitterAPIWithOAuthConsumerKey: ( NSString* )consumerKey
                                    consumerSecret: ( NSString* )consumerSecret;

+ ( STTwitterAPI* ) twitterAPIWithOAuthConsumerKey: ( NSString* )consumerKey
                                    consumerSecret: ( NSString* )consumerSecret
                                          username: ( NSString* )username
                                          password: ( NSString* )password;

+ ( STTwitterAPI* ) twitterAPIWithOAuthConsumerKey: ( NSString* )consumerKey
                                    consumerSecret: ( NSString* )consumerSecret
                                        oauthToken: ( NSString* )oauthToken
                                  oauthTokenSecret: ( NSString* )oauthTokenSecret;

+ ( STTwitterAPI* ) twitterAPIAppOnlyWithConsumerKey: ( NSString* )consumerKey
                                      consumerSecret: ( NSString* )consumerSecret;
```

上面的代码片段只是用于演示，Objectwitter-C API 是对 Twitter API 的全面封装，所以你可以查阅头文件和 [Twitter API 文档](https://dev.twitter.com/overview/documentation)来了解任何你想要的东西。

## OAuth Consumer Tokens

在 Twitter API 1.1 中，每个客户端应用必须使用 `consumer key` 和 `consumer secret` 验证自身。你可以在这个 Twitter 站点中为你的 app 申请 consumer tokens：<https://dev.twitter.com/apps>.

## Demo/Test Project

在 Objectwitter-C 项目中，在 [demo_osx](https://github.com/Twipoker-Project/Objectwitter-C/tree/master/demo_osx) 目录下包含了一个 OS X 上的 demo，这个 demo 演示了一部分 API 的用法，并且还可以允许你选择如何获取 OAuth Tokens（如下图）：

![osx_demo1](https://github.com/Twipoker-Project/Objectwitter-C/raw/master/Art/osx.png)

一旦你成功获取了 OAuth Tokens，你就可以马上使用这个简易的客户端获取你的时间轴并且发一些推文了。同时，项目中还有一个 [iOS 的 demo](https://github.com/Twipoker-Project/Objectwitter-C/tree/master/demo_ios)：

![ios_demo1](https://github.com/Twipoker-Project/Objectwitter-C/raw/master/Art/ios.png)

## 更多细节

* 浏览头文件和 Twitter API 文档
* 浏览 [Objectwitter-C 源码](https://github.com/Twipoker-Project/Objectwitter-C)或[项目的 README.md](https://github.com/Twipoker-Project/Objectwitter-C/blob/master/README.md)

## 联系作者/Troubleshooting

如果遇到任何问题，可以以下面的方式联系我：

* Find me on Twitter: [@NSTongG](https://twitter.com/NSTongG)
* Email me: `VG9uZy1HQG91dGxvb2suY29t` (Base64ed)

