之前已经来了好几篇和RSA加密相关的文章了，这次还是趁热打铁，看一下RSA在APK签名中的应用，最后我们分析一下为什么这种方式能够具有安全性。RSA对apk签名的体现就在apk文件中的META-INF文件夹中，我们先来拿一个例子分析一下。

以最新的QQ6.6.2的apk为例，现在的解压工具默认就可以解压apk了，所以也不要先改成zip然后解压了，解压后apk里面的文件大概就是这样的：
![image-20191218155802793](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191218155802793.png)

里面有一个文件夹名字叫做META-INF，这个文件夹里的东西就是今天要了解的全部内容了。先看一下里面有什么：
![image-20191218155832669](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191218155832669.png)

可以看到，里面有三个文件，当然了，如果你用到了其他的一些技术，可能里面不止这三个了。需要说明的是，MF的名字是确定的，就是MANIFEST.MF，其他的两个文件默认的文件名是CERT，但是呢，这个名字可以随意修改，只要SF和RSA的文件的名字一样就可以了，所以这里QQ的名字是ANDROIDR。我们来看一下这三个文件分别是什么作用。先来看MANIFEST.MF。

打开这个文件(记事本就可以)可以看到内容大约是这个样子：
![image-20191218155935979](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191218155935979.png)

看到第一行是指明是manifest文件，第二行是由谁创建的，然后下面的是关键内容。可以看到，这个文件中除了第一二行外，其余的部分都长的差不多都是这样的格式：

![image-20191218160041687](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191218160041687.png)

一个名字，一个特殊的字符串。Name就是文件名，SHA256-Digest就是这个文件的SHA256摘要值的Base64表示值。里面所有的这种格式内容包含了apk包中除了META-INF文件夹外的所有的文件的文件名和对应的SHA256摘要Base64值。

我们可以拿其中一个进行验证，就拿第一个来说。第一个文件的名字是AndroidManifest.xml，我们找到这个文件：
![image-20191218160129468](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191218160129468.png)

在百度上搜一个工具“在线文件sha256”，如果你懒，可以直接用我搜到的结果：

http://www.atool.org/file_hash.php

把刚才那个文件拖进去查看它的SHA256值：


可以看到文件的SHA256值为75d178e16ebe80ed754a9700ac87401eb78d78b897566a5b7a9f7f8295449038，继续看它的Base64的值，在百度上搜索hex base64，如果你懒，直接用我搜到的结果：

http://tomeko.net/online_tools/hex_to_base64.php?lang=en

将刚才的SHA256值填进去计算一下：


可以看到结果是ddF44W6+gO11SpcArIdAHreNeLiXVmpbep9/gpVEkDg=，可以看到和我们看到的MANIFEST.MF里的那个值是一样的。其他的内容读者可以自己验证。

如果计算有误，那可能是你的解压方式有问题，记得要直接解压，不要用apktool等工具解压。
再说一遍结论：MANIFEST.MF文件保存了我们apk里的除METE-INF外的文件的摘要信息。

再看第二个文件ANDROIDR.SF（默认叫CERT.SF）。

打开文件查看内容：
![image-20191218160420365](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191218160420365.png)

是不是特别眼熟，格式几乎和MANIFEST.MF一样。前面四行目前我们需要注意的是SHA256-Digest-Manifest，这个的值是我们刚才的MANIFEST.MF的文件的SHA256-Base64的值，读者自行校验。其余的部分和MANIFEST.MF一样，都是Name和SHA256-Digest，而且Name也是一样的，就是SHA256-Digest的值不一样，这个是什么含义呢？实际这个的值是对应的MANIFEST.MF里的Name和SHA256-Digest的SHA256-Base64值。没有明白？例如上面的R/o/lbs.xml的值为9C9DPqgNa7HLHjnqFy6QIC+iHOI=，这个值是MANIFEST.MF里的：



再加上两个CRLF组成的，就是说多两个回车换行。这里需要注意的是我们在验证的时候把上面的两句话保存在文本文件中，然后千万不要在记事本里添加回车换行，可以试试AndroidStudio里添加两个回车换行，这样按上面的步骤计算SHA256和Base64后就能看到结果了。

结论就是：SF文件里保存的是MANIFEST.MF文件的SHA256-Base64的值和除META-INF外所有文件的SHA256摘要Base64值的SHA256摘要Base64值。

接下来看ANDROIDR.RSA（默认叫CERT.RSA）。

这下我们就不能直接看文件内容了，因为这个RSA文件里包含了公钥和私钥签名后的一些信息。我们用下面的命令来查看一下RSA文件的内容：

openssl pkcs7 -inform DER -in ANDROIDR.RSA -noout -print_certs -text


他的基本格式是这样的：

可以看到有有效期等等信息，RSA公钥用的是1024位的，并且签名加密算法是sha256WithRSAEncryption。我猜测最下面的信息应该就是SF文件的SHA256摘要信息的私钥签名值，为什么是猜测？因为这个我还没验证出来。不过虽然没有验证，但是猜测应该是基本确定的，因为看内容知道它是128byte的，我们知道RSA加密的结果是和公私钥长度一致的，并且加密的内容不能超过公私钥的长度，而SHA256是的结果是20byte的，没有超过这个长度。从上面的命令可以看出这个RSA文件是pkcs7格式的（Android里常见的好像就pkcs7和pkcs12），这个格式我也不是特别了解，所以不敢乱说了。

先留下这个结果，可能以后有时间了再来验证一下：

94a9b80e80691645dd42d6611775a855f71bcd4d77cb60a8e29404035a5e00b21bcc5d4a562482126bd91b6b0e50709377ceb9ef8c2efd12cc8b16afd9a159f350bb270b14204ff065d843832720702e28b41491fbc3a205f5f2f42526d67f17614d8a974de6487b2c866efede3b4e49a0f916baa3c1336fd2ee1b1629652049

如果我们想直接拿到公钥，还可以这样来干：

> \> openssl pkcs7 -inform DER -print_certs -out cert.pem -in ANDROIDR.RSA
>
> \> cat cert.pem

这样会直接生成pem文件，直接查看pem文件就是公钥的值了，这里公钥的值为：



文本记录一下：

```
            -----BEGIN CERTIFICATE-----
    MIICUzCCAbygAwIBAgIES7sDYTANBgkqhkiG9w0BAQUFADBtMQ4wDAYDVQQGEwVD
    aGluYTEPMA0GA1UECAwG5YyX5LqsMQ8wDQYDVQQHDAbljJfkuqwxDzANBgNVBAoM
    BuiFvuiurzEbMBkGA1UECwwS5peg57q/5Lia5Yqh57O757ufMQswCQYDVQQDEwJR
    UTAgFw0xMDA0MDYwOTQ4MTdaGA8yMjg0MDEyMDA5NDgxN1owbTEOMAwGA1UEBhMF
    Q2hpbmExDzANBgNVBAgMBuWMl+S6rDEPMA0GA1UEBwwG5YyX5LqsMQ8wDQYDVQQK
    DAbohb7orq8xGzAZBgNVBAsMEuaXoOe6v+S4muWKoeezu+e7nzELMAkGA1UEAxMC
    UVEwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBAKFel1Yhb2lMWRXgtSkJUlQ2
    fE5k+u/weuE0iNlGYVpY3cMaQV9xfQGe3G0wuWA9Pip7PeCrfgz1Lf7jk3O8Ry+p
    lwJ9eY1Z+B1SWmns8Vbohf0eJ5CSQ4ayIwzJDjt63JVgPdz0xAvccvItsPIWqZw3
    HTv4nLpleMYGmeig1TaVAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAlKm4DoBpFkXd
    QtZhF3WoVfcbzU13y2Co4pQEA1peALIbzF1KViSCEmvZG2sOUHCTd86574wu/RLM
    ixav2aFZ81C7JwsUIE/wZdhDgycgcC4otBSR+8OiBfXy9CUm1n8XYU2Kl03mSHss
    hm7+3jtOSaD5FrqjwTNv0u4bFillIEk=
            -----END CERTIFICATE-----
```

如果我的猜测没有错误的话，用这个公钥解开那个被私钥签名的值获得的结果应该就是SF文件的SHA256值了。

最后总结一下apk签名的整个流程：

一、对Apk中的每个文件做一次算法(数据SHA256摘要+Base64编码)，保存到MANIFEST.MF文件中

二、对MANIFEST.MF整个文件做一次算法(数据SHA256摘要+Base64编码)，存放到CERT.SF文件的头属性中，在对MANIFEST.MF文件中各个属性块做一次算法(数据SHA256摘要+Base64编码)，存到到一个属性块中。

三、对CERT.SF文件做签名，内容存档到CERT.RSA中

整体基本就是这个样子，现在补充一些上面没有说到的，为什么RSA和SF文件名字可以随意指定，只要一致就可以，稍微看一下源码就知道了：

key.endsWith(".DSA") || key.endsWith(".RSA") || key.endsWith(".EC")

上面的是我从代码里拷出来的一个if语句的条件，可以看到，在apk安装验证的时候找RSA文件并不是通过名字找的，而是通过后缀找的。同样我们可以看到，apk的签名算法不只是支持RSA，实际DSA和EC也是支持的，并且它们混合起来用也是行的（外面有个循环语句，我没有贴出来）。
接着来看一下和SF相关的代码：

String signatureFile = certFile.substring(0, certFile.lastIndexOf('.')) + ".SF";

可以看到，找SF文件是通过RSA文件的名字来找的，所以需要SF和RSA的文件名字一致。对于SF文件的内容有两点需要补充，

其一是有时候SF文件中可能会有SHA256-Digest-Manifest-Main-Attributes这样的值，它的含义是META-INF目录下MANIFEST.MF文件内，头属性块的hash值。

其二是这个文件可能内容和我们说的不一样，例如：

可以看到MF文件和SF文件的下部分内容居然一样！！！这个结果让当时的我疑惑不解，甚至想要放弃本篇内容的研究，因为我在网上搜了很多应用，结果都不是我看到的这个样子，难道我的是特殊的？疑惑了好几天，不过经过了一段时间的折磨，终于还是稍微明白了些。我查看了我们公司的历史apk包，二分查找对比，最后发现在SF文件中有写Android Gradle 2.2.2的包就会导致MF和SF下部分一样（看来我们公司还是挺超前的）。最后在网上终于找到了一个相关内容，那就是Android Gradle Plugin在2.2.0时的打包机制变化了！具体的变化我用两句话分别概括改变和原因：

新签名验证的是整个apk的二进制文件，不同于之前的每个文件来验证完整性，一个apk可以同时支持新旧两种方法的签名，所以它保持了向前的兼容性。
原因是1安全，验证整体二进制，不再可以修改压缩包里的文件；2速度，安装无需解压缩，缩短安装时间。

可能就是这个新的签名机制导致了MF和SF文件的内容相同，所以我们上面的结论只是用于Android Gradle Plugin在2.2.0之前。

对于RSA文件，需要说明的是本例中的公私钥的长度是1024位，实际你可能看到的是2048位的了，并且签名加密算法本例是sha256WithRSAEncryption，实际可能你看到的是sha256WithRSAEncryption。

补充就这么多，apk安装验证的过程实际就是和这个过程正好相反嘛，简单的说一下就是：

找到RSA文件用公钥解密私钥的签名后的信息，如果能解密，这一步通过；

解密后的值和SF的SHA256值进行比对，如果一致，这一步通过；

查看SF文件中的MF文件的SHA256-Base64值，如果和MF的计算值一样，这一步通过；

计算MF中的Name/SHA256-Digest属性块的SHA256-Base64值和SF里的对比，如果一致，这一步通过；

计算除META-INF外的每个文件的SHA256-Base64值和MF里的对比，如果一致，这一步通过。

以上基本就是apk安装时的验证过程，当然如果是覆盖安装就是多一个操作，那就是身份的验证，如果想要覆盖，那么必须包名一致，签名一致，这里的签名一致实际说的就是公钥需要一致。

接下来才是本文的最终目的，分析一下这样做的必要性，我们通过反证法来说明。

假如我们是一个非法者，想要篡改apk内容，我们怎么做呢？如果我们只把原文件改动了（比如加入了自己的病毒代码），那么重新打包后系统就会认为文件的SHA256-Base64值和MF的不一致导致安装失败，既然这样，那我们就改一下MF让他们一致呗？如果只是这样那么系统就会发现MF文件的内容的SHA256-Base64与SF不一致，还是会安装失败，既然这样，那我们就改一下SF和MF一致呗？如果这么做了，系统就会发现RSA解密后的值和SF的SHA256不一致，安装失败。那么我们让加密后的值和SF的SHA256一致就好了呗，但是呢，这个用来签名加密的是私钥，公钥随便玩，但是私钥我们却没有，所以没法做到一致。所以说上面的过程环环相扣，最后指向了RSA非对称加密的保证。有人说，那我可以直接重签名啊，这样所有的信息就一致了啊，是的，没错，重签名后就可以安装了，这就是说签名机制只是保证了apk的完整性，具体是不是自己的apk包，系统并不知道，那我们上面说的安全性是怎么保证的呢？那就是我们可以随便签名，随便安装，但是在覆盖安装的时候由于我们的签名和作者的签名不一致，导致我们重签名后的apk无法覆盖掉原作者的。这就保证了已经安装的apk的接下来的安全链的正确性。当然了，如果你的手机上来就直接安装了一个第三方的非法签名的apk，那么原作者的官方apk也不能再安装了，因为系统认为他是非法的。

最后，上面说了，无法做到修改apk后重签名来覆盖原作者的apk，那么如果手机上本来就没有原作者的apk包呢，单独给我们一个apk我们能玩出什么花样呢？这些留在以后吧，预告一下之后的可能的内容：反编译是怎么搞呢？二次打包如何搞呢？怎么植入代码呢？等等等等。

参考：https://blog.csdn.net/lostinai/article/details/54694564