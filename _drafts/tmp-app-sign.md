
app签名的流程：

1. 首先需要生成一个开发者证书，具体是去开发者网站进行操作，期间会在本机手动创建一个`CertificateSigningRequest.certSigningRequest`文件，用他生成了一个`.cer`证书。

这一步又涉及到以下步骤：

1.1 从keychain中进行“从证书颁发机构请求证书”，也就是CSR文件。这个文件是用来给CA创建证书。具体是生成了一对公私钥，公钥存在了`.certSigningRequest`文件中，私钥则保存在本机电脑中；

1.2 网站中使用到`.certSigningRequest`文件，这个时候首先会生成一个签名，将文件中的公钥hash后用苹果后台的私钥去加密。随后将文件中的公钥和生成好的签名放在一起，也就是我们得到的cer证书。

可以通过 `security find-certificate -c 'name' -p` 查看证书内容。

2. 接下来就是对app进行签名了，对app进行签名是用的本地的私钥，也就是之前创建CSR时候生成的一份保存在本机的私钥。

对app签名使用的是`codesign`这个命令，会对app里所有文件(各种图片音频资源，framework)除可执行文件进行计算hash(单纯的hash没有用私钥加密)，保存在`_CodeSignature/CodeResources`中。app签过名的可执行文件会多一条load commands，`LC_CODE_SIGNATURE`，可执行文件中对应有`Code Signature`，一段二进制的数据，所以签过名的二进制文件体积会变大。

> 对framework签名和对app签名一样，首先对framework里的资源进行hash，存放在`_CodeSignature/CodeResources`中，其次framework中可执行文件中有`LC_CODE_SIGNATURE`load command，`Code Signature`数据段。

`Code Signature`中最重要的是CodeDirectory部分，是整个MachO文件的hash，具体是将MachO文件按4K大小进行分页，对每页hash，按照顺序保存下来。同样的和文件一样依然没有进行加密，因为非对称加密的性能不太好，所以只需要保证这些hash没有被篡改就好。在Maco0文件开始页的前面，存在几个特殊的页，存储了Info.plist，CodeResources，Entitlements等文件的hash。综上，CodeDirectory是一个分页的机构，存储了app里一些资源的hash以及二进制文件的hash。

`Code Signature`中还有CMS Signature，这段签名中除了存储了证书的信息，还存储着签名信息。结构如下：

```
CMS Signature:
	Content:
		certificates; //证书链
		signerInfos；// 签名信息
			signedAttrs；//对 `CodeDirectory`计算的hash值，叫做CDHash
			signature； // signedAttrs计算hash，然后用私有加密的签名
```

可以看到只有CMS Signature 中的signature是用私钥加密的签名，其他的都只是求了一个hash而已。

```
这里需要注意的是，当我们从获得cer证书后，keychain会将这个证书和CSR进行关联，选择某个证书进行签名的时候，keychain会找到对应CSR的私钥进行签名。

可以看到私钥和设备是绑定的，如果另一台设备也需要怎么办？这个时候可以导出，也就是`.p12`格式的文件，这样其他Mac也可以用这个证书了。
```

接下来就是打包步骤，默认的我们需要将CA也就是苹果颁发的证书给打包到App中。iOS设备存有对应苹果后台私钥的公钥，可以对证书进行验证。验证证书有效过，会用证书中的CSR公钥进行验证app签名，这样就间接验证了app安装通过了Apple的允许。

实际安装过程中会有一些限制，比如Ad Hoc的ipa有设备的限制，必须是注册的设备才能安装；以及AppId的限制。

所以打包到App中除了证书还需要其他的东西，注册的设备id，以及AppId，Entitlements，这些额外的东西加上之前的证书信息，苹果用Provisioning Profile进行存放。

可以通过`security cms -D -i path` 查看provision内容。

一个Provisioning Profile包含了证书的信息，deviceId，AppId，Entitlements，前面所有信息的签名。这样设备验证的时候，首先用公钥验证签名，然后得到了DeviceId等信息进行验证，下面用公钥验证证书(证书是苹果服务器用私钥加密过的，用iOS设备对应的公钥进行验证)，最后用证书的公钥(CSR创建的一对公私钥)验证App签名。

当app在运行时，当页表中某个表项指向的是磁盘地址，这个时候会触发缺页中断，这个时候内核会对该页进行签名校验，也就是计算该页的hash和CodeDirectory中记录的值进行比对。

# 重签名

根据上面的知识，我们可以做到重签名。如果包是app store下载下来的，需要一个所谓的砸壳操作。因为当我们把ipa上传到App Store后，会进行加密。

这个时候使用iOS App Signer这个工具，指定证书以及Provisioning Profile文件即可重签名。

这个工具首先会用证书对app里对framework进行重签名。第二步是对App进行重签名，本次签名需要指定entitlements，`codesign -f -s "证书名称或者SHA1值" --entitlements resign.entitlements Target.app`。 而resign.entitlements 可以根据`.mobileprovision`来生成，该文件中包含entitlements的信息。

# References

[http://xelz.info/blog/2019/01/11/ios-code-signature/](http://xelz.info/blog/2019/01/11/ios-code-signature/)

[https://www.sslshopper.com/what-is-a-csr-certificate-signing-request.html](https://www.sslshopper.com/what-is-a-csr-certificate-signing-request.html)

[https://blog.cnbang.net/tech/3386/](https://blog.cnbang.net/tech/3386/)

[https://www.apple.com/business/site/docs/iOS_Security_Guide.pdf](https://www.apple.com/business/site/docs/iOS_Security_Guide.pdf)

[https://www.objc.io/issues/17-security/inside-code-signing/](https://www.objc.io/issues/17-security/inside-code-signing/)
