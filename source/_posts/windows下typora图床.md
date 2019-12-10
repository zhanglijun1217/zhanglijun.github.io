---
title: windows下typora图床（附带阿里云教程）
copyright: true
date: 2019-12-10 00:43:56
tags:
	- windows
	- typora
	- 图床
categories:
	- 日常工具
---

# typora

Typora是大家写博客、记笔记、写文档等日常使用场景下都会使用的一个MarkDown语法的软件，对于熟悉markdown语法和喜欢markdown简洁性的朋友来说，typora是不可或缺的工具。但是，对于图片处理，我们需要图床去将我们的本地图片（截图、流程图之类的）上传到第三方的对象存储上（当然自己的服务器也是可以的）。

本文基于一个typora在windows下的小插件[windows下typora图床]( https://github.com/Thobian/typora-plugins-win-img ) 来实现实时的粘贴图片到typora即将你的图片上传到阿里云OSS上，并且替换的实现过程，github上已经其实写的比较明白了，但是还是想把自己接入的过程和踩得坑记录一下。

对于typora的使用、阿里云OSS的使用（我记得一年只要个位数的钱）、markdown语法等网上有很多介绍和例子，下面是几个传送门：

1. [阿里云对象存储]( https://help.aliyun.com/document_detail/31947.html) 

2. [typora.io]( https://www.typora.io/ )

3. [markdown语法]( https://www.markdownguide.org/basic-syntax/ )

# typora图床

## 怎么发现的

如果你没有图床，那么在你写博客的过程中如果要使用阿里云图片的外链，得是这样的操作。

1. 将图片上传到阿里云OSS对象存储上

![image-20191210201936023](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210201936-475640.png)

2. 复制该图片的外链

![image-20191210202043284](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210202043-187642.png)

3. 将外链用markdown语法粘贴到正文中

![image-20191210202125030](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210202126-882435.png)

这真的真的相当麻烦。_(:з」∠)_

所以图床就是用来解决这个问题，但是之前用的图床（之前用过chrome的一个图床插件）都是也只是省去了你登录对象存储在页面上上传的这一步，最后就还是要复制生成的url然后到typora的文章中。

这里就在网上搜了下typora的图床，看看有没有符合自己偷懒的想法的做法，一键截图之后复制到typora中然后就可以了。

然后就是Google搜了下，发现第一条就是日常学（划）习（水）的网站——知乎。

![image-20191210202739755](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210202740-488773.png)

于是就点开之后看到了typora的这个插件，进而有了这个文章

## 手工教程

首先贴一下这个插件的地址：[github]( https://github.com/Thobian/typora-plugins-win-img )

这个文档中和知乎的回答差不多，我们可以直接来到使用这里开始：

![image-20191210203153008](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210203153-947469.png)

下载插件的代码到本地，这里不熟悉的github的同学可以直接点击图中的download zip即可。

![image-20191210203319035](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210203319035.png)

解压之后可以看到有这些文件，和github上的目录对应

![image-20191210203431182](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210203431-481659.png)

然后按照文档上的教程手工替换（复制plugins目录、替换window.html）到对应的typora安装目录下的resource\app目录下。替换完成之后的目录长这个样子。

![image-20191210203640933](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210203641-860142.png)

然后就是对里面的代码进行自己OSS的配置了。

打开plugins-->image-->upload.js文件（这里可以直接用记事本打开js文件，当然程序员自动sublime或者vscode。），拉到底就可以看到文档中说的init相关的代码。

![image-20191210203936380](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210203936-497519.png)

然后复制文档上的阿里云这段配置，把整个473行替换掉。

![image-20191210204100544](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210204100544.png)

接下来就按照注释（“//“后面的东西）来操作即可。

首先插件作者建议你添加一个子账号来单独操作你的OSS，这里简单理解下就是在你的阿里云上你可以建立多个用户组，而每个用户组中可以建立多个子用户，通过对用户组或者用户来设置权限达到一定操作。插件其实是用代码去调用阿里云提供的API来操作上传图片的，所以你肯定要在本地的typora的配置代码里填上一个关联你OSS的账号并且配置对应的权限才能成功上传图片；同时，出于安全考虑，你的这个账号应该只对你的OSS有写入和读的操作权限，所以建议来个子账号专门搞这个事情。

作者其实这里写的也很明白了，包括申请子用户的地址：`https://ram.console.aliyun.com/users`

打开之后点击新建用户，即可看到让你填用户账户信息。
![image-20191210204747378](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210204749-103389.png)

当然这里也可以直接添加用户组，然后在组下面添加用户。

填写你想要的登录名称（复杂点也没关系，在后面配置一般是关键字搜索选择的）、显示名称和勾选上**编程访问**。这里的编程访问我们也可以清楚看到是通过assess信息来支持开发者调用API访问的用户。

![image-20191210205026239](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210205026239.png)

点击确定，这时要收一个验证码：

![image-20191210205203529](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210205203529.png)

填写完之后这步很关键，可以看到会在页面上告知你一个accessKeyId和accessKeySercret，但是坑的地方是这个授权key的值只有在创建的这个页面才能看到，之后就看不到了，所以这里一定要进行复制或者保存这两个值。可以看到页面上也提供了对这两个值的复制功能。

![image-20191210205455114](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210205455114.png)

这时候就要给这个用户去设置权限。刚创建的子用户是没有任何权限的，这里你既然要对OSS进行上传写入的操作，肯定要给这个账户权限，可以看到在用户界面有拟刚才创建的子用户，并且可以配置权限

![image-20191210205719451](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210205723-594012.png)

这里在权限搜索框内输入OSS，就可以看到我们要配置的权限（管理OSS的权限），点击确定即可。

![image-20191210205859885](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210205901-495252.png)

到这里，init代码中需要的前三项就都有了。

![image-20191210210321356](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210210321-914840.png)

还剩个bucketDomain，这里作者其实也说的很明白了，在你的OSS概览页面上，会有对应的bucket域名，这里挑选一个即可，我挑选的是外网访问的这个bucket域名

![image-20191210210533111](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210210533111.png)

其实文档到这步也就没有了，我就以为是OK了，然后就很有自信的去保存了修改后的js文件去试了下。果然，不行_(:з」∠)_。

一直在提示我`服务响应解析失败，错误`。之后又对了遍文档发现也没漏啥。最后还是选择去看了upload代码。

首先这个错误肯定是作者定义的，然后就在upload.js中看到了这个错误。

![image-20191210210949483](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210210950-512239.png)

这里可以看到其实是调用接口异常了，所以返回的这个错误，所以大概率是刚才配置的问题，再往上翻才看到原来除了文件底部的init方法之外，还要去配置下代码中的setting信息，这里稍微吐槽下为啥文档里没有写（没有，就是我前端不熟就没看代码，菜是原罪= =）。

在setting配置里有这段代码：
![image-20191210211346838](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20191210211346838.png)

可以看到请求的域名和API访问是从这里解析的，所以你不配置这里，默认会用作者写死的jiebianjin去访问接口，当然调不通了（因为阿里云上并没有这个子用户）。这里还是刚才的accessKey信息和bucket信息。这里看到了设置了过期时间和上传的大小限制，进而手工改了下大小限制，默认是512k，我这里放大到了5M。

之后保存之后，又信心满满的去试了下，卧槽，居然还是报错。然后又对了下文档中的配置。

无奈打开了调试，typora其实就是个浏览器= =。windows下shift+f12是调试工具。这里没有把当时报错的接口的截图贴上，当时忘记截图了，这里是我在写这篇博客的时候调试截图的。通过一顿分析发现自己在配置BucketDomain的时候，忘记在最后加上一个反斜杠。这里大家在配置的过程中也可以注意下。

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210212015-416701.png)

![image-20191210212848266](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210212848-345879.png)



## 给插件的建议

首先感谢插件作者 @Thobian 大佬写了这个插件，真的在typora下很方便，应该以后会比较重度使用。这里提几个建议。

- 完善各个厂商对象存储的配置教程，其实腾讯云OSS我也用过，和阿里云大同小异。
- 点击再上传这个功能可以加个弹窗之类的中间过程（不能省这个我get到，因为可能失败要重传），在写文过程中点击到图片会自动再上传一次，OSS上会有比较多的重复文件，不好管理
- 接第二条，我在写文过程中喜欢`ctrl+s`保存，这个好像也会把我的图片再重新上传一次（不太确定），也会造成大量的相同文件。
- 性能上有时会卡顿，如果是阿里云接口的锅当我没说哈_(:з」∠)_。
- 继续开发更好的功能给大家用，会一如既往的支持的~

![image-20191210211918104](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/typora/20191210211918-197363.png