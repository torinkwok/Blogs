文章中提到的环境或工具：

* OS X Mavericks
* Apple Keychain Access Utility 9.0
* Mozilla Thunderbird 31.3.0
* Apple Mail 7.3
* Google Chrome 39.x

相较于使用 GnuPG 收发加密邮件而言，使用 S/MIME 证书对邮件进行加密和签名要更加轻量级，因为大多数主流的邮件客户端都内置了对 S/MIME 的支持，而对 GnuPG 的支持还要额外安装插件。

## 向 CA 申请 S/MIME 证书
如果你已经有了一个由受信任的 CA （如 VeriSign 或 DigiSign等）签发的 S/MIME 证书的话，可以跳过这一节。另外使用 Keychain Access 工具创建的 self-signed 证书只能用于签名邮件，而不能用于加密。

在开始使用 S/MIME 加密和签名邮件之前，需要做的一件事就是向证书颁发机构（Certificate Authority）申请一个 S/MIME 证书，有一些商业的证书签发机构提供为个人用户免费签发 S/MIME 证书的服务，如 Comodo Limited 等，在这篇文章中，我将使用由 Comodo Limited 签发的 S/MIME 证书。

可以在[这里](https://www.instantssl.com/ssl-certificate-products/free-email-certificate.html)免费申请一个 S/MIME 证书。

![get-s/mime-cert-now](http://i.imgbox.com/9BqciyfV.png)

点击`Get Now`按钮，填充该表单：

![fill-out-form](http://i.imgbox.com/aEJsZFi9.png)

表单中需要你填写项屈指可数，`First Name`，`Last Name`，`Email Address`，`Country`，这几个项都是会出现在为你签发的证书的属性（attributes）中，所以要认真填写。

用过 GnuPG 这类加密软件的人可能会很熟悉 `Revocation Password` 这一项，在 GnuPG 中，当你认为你的私钥（private key）被泄露时，可以对自己的公钥（public key）执行吊销（revoke）操作，执行该操作后，拥有你的公钥的人在从公钥服务器上刷新了状态后，就会看到你的公钥已被吊销（通常会附上吊销原因），这样，他们就会知道，该公钥以被泄露，已不安全。S/MIME 证书的 revocation password 也由此作用，在日后如果你认为你的证书的安全数据（secret，如私钥）已被泄露，可以对其进行吊销，为了确保只有你自己能够执行吊销操作，就需要该吊销密码。__吊销密码一定要采用和其他服务完全不同的密码。__

最后，推荐不要选中名为 `Comodo Newsletter` 的 check box，商业公司的广告邮件很烦人的，你懂得。

Okay，一切确认无误后，点击 `Next`，准备 collect 你自己的 S/MIME 证书。

点击 `Next` 按钮后，可能会出现类似的页面：

![user-defined](http://i.imgbox.com/jJVqoJk2.png)

不用担心，刷新浏览器，Chrome 会询问你“是否确认重新提交表单”，选择 `Continue` 即可。

成功提交表单后，会看到该页面：

![application-is-successful](http://i.imgbox.com/9r0ktFrA.png)

证书已经申请成功，下载方式被发送到了你在刚才那个表单中所填写的 Email 中，检查该 Email 的 inbox，可以看到一封标题为 “Your certificate is ready for collection!” 的邮件：

![email](http://i.imgbox.com/NKVMXZCU.png)

点击那个大大的红色的 `Click to Install Comodo Email Certificate` 按钮，这时就会跳转回你的浏览器，可以看到：

![collection-of-secure-email-cert](http://i.imgbox.com/cpaHhAzv.png)

“已经成功地存储了 __COMODO RSA Client Authentication and Secure Email CA__ 颁发的客户证书”，这说明系统已经自动地将为你签发的 S/MIME 证书存储到了本地的证书数据库中，在 OS X 上，证书会被存储到 Keychain Services 中，可以打开 Keychain Access，在你自己的默认的钥匙串中（OS X 上默认名为 login）可以看到刚刚签发的 S/MIME 证书：

![keychain](http://i.imgbox.com/2DIGcxKM.png)

![stored-cert](http://i.imgbox.com/CFZhfgE6.png)

## 为 S/MIME 证书构建信任链（trust chain）

你可能已经注意到了，刚刚申请的 S/MIME 证书的评估（evaluate）结果是 __“The certificate was signed by an unknown authority”__：

![The certificate was signed by an unknown authority](http://i.imgbox.com/VnTF017J.png)

这是由于**用于签发该 S/MIME 证书的中间证书（intermediate certificates）或根证书（root certificates）不在该证书链（certificates chain）之中**，从而无法构建完整的[信任链（trust chain）](https://en.wikipedia.org/wiki/Chain_of_trust)。

在无法构建完整的信任链的情况下，是无法使用该证书加密邮件的。

**如果你的证书评估结果为 “This certificate is valid”，那么说明构建信任链的必要的中间证书和根证书都存在于当前钥匙串中，那么就可以跳过这一节。**

构建该 S/MIME 证书作为 leaf 节点的信任链至少需要三个中间证书，分别是：

* [COMODO RSA Certification Authority](http://crt.comodoca.com/COMODORSACertificationAuthority.crt)

* [COMODO RSA Client Authentication and Secure Email CA](http://crt.comodoca.com/COMODORSAClientAuthenticationandSecureEmailCA.crt)

* [COMODO Client Authentication and Secure Email CA](http://crt.comodoca.com/COMODOClientAuthenticationandSecureEmailCA.crt)

同时还需要一个根证书：

* [Add Trust External CA Root](http://crl.usertrust.com/AddTrustExternalCARoot.crl)

点击相应的链接即可下载这些证书，双击后即可自动将其添加到当前钥匙串中。一般 `Add Trust External CA Root` 证书会预先安装在系统根（System Roots）中，你可以在 Keychain Access 中搜索系统根证书列表，如果没有该证书，下载，添加。

为了确保万无一失，推荐将上述四个证书全部下载并安装，如果钥匙串中已经存在了该证书，那么安装动作会被忽略。

当确定安装了这四个证书后，可以看到你刚刚申请的自己的 S/MIME 证书的当前评估结果变为了**绿色**的 “This certificate is valid”，同时还显示了整个证书链的结构：

![now-is-valid](http://i.imgbox.com/AA026cHK.png)

这说明已经为你的 S/MIME 证书构建了完整的证书链，它已经准备好开始自己的工作了。

## 在 OS X 自带的 Mail 中使用 S/MIME 证书加密和签名邮件

当构建了完整的信任链后，即可开始使用该 S/MIME 证书加密和签名邮件。

这一节演示了在 Apple Mail 中使用 S/MIME 证书对邮件进行加密和签名，如果你是 Thunderbird 用户，并且对 Apple Mail 没有兴趣，可以跳过这一节。

OS X 自带有一个 Mail 应用（由于 Mail 名称太过简洁，容易造成歧义，所以在下文中都采用 Mail.app 的名称指代），Mail.app 内置了对 S/MIME 加密的支持，使用 Mail.app 收发 S/MIME 加密邮件的好处就是：非常容易，无须进行任何额外的配置。但代价就是可定制行极低，因为 Mail.app 的偏好设置（Preferences）面板中甚至都没有与 S/MIME 相关的设置项。

Anyway，对于非技术背景同时又想获得加密邮件带来的好处的用户来说，Mail.app 无疑是最好的选择。

注：因为在 Mail.app 中我还没有成功配置 openmailbox.org 的邮箱，所以在这一节中，我将使用 esquiiire@gmail.com 这个邮箱，和使用该邮箱申请的 S/MIME 证书作为演示。sender 邮箱为 esquiiire@gmail.com，receiver 邮箱为 Tuhn-Kuo@outlook.com。

在 Mail.app 中新建邮件，像往常一样撰写一封邮件：

![osx-mail.app-1](http://i.imgbox.com/LjM8YV76.png)

在这里你可以主要到，红框圈上得三个控件都是灰色的禁用状态（disabled）。这是因为，当前默认的钥匙串中还没有与发送者（esquiire@gmail.com）匹配的 S/MIME 证书，Mail.app 就是根据 S/MIME 证书中的邮箱地址来自动识别在加密邮件时应该使用那个证书的，而此时没有匹配的证书，所以无法加密。

现在把我的 S/MIME 加入到钥匙串后（在 OS X 中双击证书文件后即可将其加入到钥匙串中），***重启 Mail.app （这是很必要的一步）***，可以看到，S/MIME 开关变为了蓝色的可用状态（enabled），而加密和签名两个按钮也变为了可用状态：

![osx-mail.app-2](http://i.imgbox.com/XRTgbmA0.png)

这里必须要注意的一点就是，用来加密和签名邮件的 S/MIME 必须同时具有公钥和私钥，也就是说必须是你自己的 S/MIME 证书，而对于那些只有公钥没有私钥的证书（比如你的朋友给你的他的公钥证书），是无法用来加密邮件的。

同时，如果想能够加密发给对方的邮件的话，必须要在钥匙串中拥有对方的 S/MIME 公钥证书，可能是你的朋友导出（export）后给你的，或者是从互联网上获取的。如果没有对方的公钥证书，那么无法进行加密，那个小锁头按钮会一直处于禁用状态，但是可以进行签名。关于这些组合，可以自己试验一下。

当对方接受到邮件后，会看到：

![osx-mail.app-3](http://i.imgbox.com/jCJAyMK4.png)

Security 状态中的 Signed 和 Encrypted 表明以同时被加密并且签名。现在你可以登录你的邮箱的 web 版看一看，是无法读取这些加密过的邮件的，只有在你所用的邮件客户端上使用自己的 S/MIME 证书的私钥才能解密这些邮件。

## 在 Mozilla Thunderbird 中使用 S/MIME 证书加密和签名邮件

现在到了最关键的部分，这部分也是想写这篇文章最主要的原因，因为笔者就是 Mozilla Thunderbird 的忠实用户，但是因为 Firefox 和 Thunderbird 在设计上的一些缺陷，为了能够在 Thunderbird 中使用 S/MIME 加密邮件，的确费了一番周折。

Thunderbird 绝对堪称最强大的邮件客户端，其内置了对 S/MIME 的支持，同时可定制性非常高。但是 Thunderbird 和它兄弟 Firefox 一样，有一个广受用户尤其是 OS X 用户诟病的问题：Thunderbird 并不使用 OS X 的钥匙串服务（Keychain Services）管理密码和证书，而是自己实现了密码和证书管理器。这么做可能是因为二者对跨平台的支持，所以自己重新实现密码和证书管理器相对要更容易，但是这也带来了很大的问题，那就是**Thunderbird 无法与 OS X 上的其他应用共享密码和证书**。

这其实就是让我“颇费周折”的原因，我掉进了这个陷阱，折腾了很久才走出来。

首先，打开当前邮箱的账号设置面板：

![thunderbird-1](http://i.imgbox.com/KxeO9dSU.png)

选中当前账号的 `Security` 项：

![thunderbird-2](http://i.imgbox.com/773fUFS9.png)

刚才已经提到了，Thunderbird 不使用 Keychain Services 管理密码和证书，所以我们就要把之前构建证书信任链中的所有步骤在 Thunderbird 自己的证书管理器（certificates manager）中重新做一遍。

选择 `View Certificates` 打开证书管理器：

![thunderbird-3](http://i.imgbox.com/lmICnthu.png)

将自己的 S/MIME 证书（首先要从 Keychain Access 中将该证书导出，这一步很简单，不赘述），然后将该证书导入到 `Your Certificates` 列表中：

![thunderbird-4](http://i.imgbox.com/3oGFG8Sz.png)

点击 `导入`：

导入成功：

![thunderbird-5](http://i.imgbox.com/46YIwk3w.png)

![thunderbird-6](http://i.imgbox.com/NT7d4jei.png)

这时还要将对方的公钥证书导入到 `People` 列表：

不过现在还无法重新导入，可以看到这个错误：

![thunderbird-7](http://i.imgbox.com/OmuBRWQk.png)

这是由于 Thunderbird 自己的 Certificates Manager 与 Keychain Access 的一个区别：正如之前所见，Keychain Access 可以自由地导入一个不被信任的证书，只不过它会在评估结果中以红色字体写成：__“The certificate was signed by an unknown authority”__，但是 Thunderbird 的 Certificate Manager 不可以，向其中导入的证书必须在当前证书数据库中拥有完整的信任链，也就是说必须能够从你的证书追溯到受信任（trusted）的根证书。

所以在导入其他证书之前，我们必须将之前提到的那3个中间证书和1个根证书导入到 Thunderbird 自己的证书管理器中：

---
切换到 `Authorities` 列表：

![thunderbird-8](http://i.imgbox.com/uyBpbRgU.png)

---

点击 `Import` 按钮，并将事先从 Keychain Access 中导出的三个中间证书导入：

![thunderbird-9](http://i.imgbox.com/2P5bpvNj.png)

---

将三个选项全部勾选：

![thunderbird-10](http://i.imgbox.com/K43AFKrr.png)

同时可以查看一下 `Authorities` 列表中 'C' 开头的项，看看能否找到 Add Trust External CA Root 根证书，如果不能找到，和上面的步骤一样，导出，导入。

现在我们已经在 Thunderbird 自己的 Certificates Manager 中为自由 Comodo Limited 签发的个人 S/MIME 证书构建了完整的信任链，可以重新导入邮件接收者的公钥证书了：

![thunderbird-11](http://i.imgbox.com/fDTskZwi.png)

成功导入。

需要记住的和 Keychain Access 的又一个区别：

在 Certificate Manager 中

* 自己的 S/MIME 证书必须要导入到 `Your Certificates`，该列表的 Import 对话框也只支持导入含有私钥的证书格式：PKCS12(.p12)；

* 对方的 S/MIME 证书（只含有对方的公钥）则只能导入 `People` 列表，该列表的 Import 对话框只支持导入仅含有公钥的证书格式：Certificate(.crt)；

* 中间证书必须导入到 `Authorities` 列表中。

Keychain Access 则没有这么多的区分，可以将任何类型的证书导入到同一个 Keychain 中。

现在就可以使用 Thunderbird 加密和签名邮件了。首先，为你的邮箱账号设置数字签名（digital signing）时默认使用的 S/MIME 证书：

![thunderbird-12](http://i.imgbox.com/ES0mIpXO.png)

选择刚刚申请的 S/MIME 证书：

![thunderbird-13](http://i.imgbox.com/iyamYrwV.png)

是否想要为加密和签名使用相同的证书？选择是：

![thunderbird-14](http://i.imgbox.com/U4x90sWg.png)

现在已经同时为加密（Encryption）设置了与数字签名操作使用的相同的证书：

![thunderbird-15](http://i.imgbox.com/AlIrVeT2.png)

至于是否设置成`默认加密`和`默认签名`，以及两者的不同组合，那就看自己的口味了。

现在像你平时使用 Thunderbird 那样撰写一封邮件：

![thunderbird-16](http://i.imgbox.com/F5FjmPrr.png)

同时选中 `Encrypt This Message` 和 `Digitally Sign This Message` 选项以同时加密和签名该邮件。加密和签名的状态会在右下角显示出来：

![thunderbird-17](http://i.imgbox.com/j2jcBc4b.png)

点击两个小图标以取消或选中。

检查 esquiiire@gmail.com 的收件箱：

![thunderbird-18](http://i.imgbox.com/FjDCCtQa.png)

右上角的两个图标表示改该邮件在被发送之前进行了加密和签名。点击它们即可查看发送者的证书信息，并且可以查看邮件的签名状态，即：邮件被签名后，是否被人恶意或无意篡改：

![thunderbird-19](http://i.imgbox.com/NG7SXQpO.png)

当你希望对方给你发送 S/MIME 加密邮件时，导出你的 S/MIME 证书的公钥部分，让对方存储在自己的证书数据库中即可。关于公钥的交换问题，在 GnuPG 中通过公钥服务器这种形式很方便地解决了，但是 S/MIME 证书在这方面的解决方法，我还不太清楚，没有尝试过。

## 在移动设备上使用 S/MIME

这篇文章是专门介绍在 OS X 上使用 S/MIME 的，关于在 iOS 上使用 S/MIME，参考[这篇文章](http://feinstruktur.com/blog/2011/12/12/using-smime-on-ios-devices.html)。

## Summary

为了测试 S/MIME 加密，我昨晚测试了15款 OS X 上的，10余款 Android 上的以及2款 Windows 上的邮件客户端。

除了 Thunderbird 以及 [Windows Essentials](http://windows.microsoft.com/en-us/windows-live/essentials) 套件中的 [Windows Live Mail](http://windows.microsoft.com/en-us/windows-live/windows-essentials-help#v1h=tab4) 之外，其他的客户端对 S/MIME 的支持都非常简陋或者根本不支持。在 OS X 上以其自带的 Mail.app 为首的邮件客户端，都属于这一类，它们的设计使得 S/MIME 几乎开箱即用，但是功能却非常简陋，简陋到甚至在 Preferences 面板中没有针对 S/MIME 的设置项，对加密邮件时的证书的选择也完全依赖于**将证书中的 attributes 的 Email 域与邮件发送者的邮箱**进行匹配，相同，就使用，不相同，就无法使用。然而在 Thunderbird 中，为 S/MIME 提供了大量定制功能，你可以根据自己的口味定制使用规则（rules），比如为一个邮箱使用与其名称不同的证书，这些在 Mail.app 中是做不到的。

文章中笔者提到的两款邮件客户端，是两种设计的方式，其他的客户端上的配置方式如出一辙。

不管怎么说，加密和签名你的邮件很重要。无论何时何地，在做什么，都要记住：**Big Brother Is WATCHING You.**

## Contact Me

如果你遇到任何问题，你可以给我发送邮件：Tong-G@outlook.com 或者在 Twitter 上 DM 我：[@NSTongG](https://twitter.com/NSTongG)

如果你使用GnuPG的话，也可以给我发送加密数据，我的GnuPG公钥为 0x67B9E95236924648，你可以从公钥服务器上 retrieve 之。

Blog: http://nstongg.tumblr.com
GitHub: https://github.com/TongG