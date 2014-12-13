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

## 在 Mozilla Thunderbird 中使用 S/MIME 证书加密和签名邮件