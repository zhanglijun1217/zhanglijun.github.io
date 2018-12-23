---

title: 记一次freemarker错误排查
copyright: true
date: 2018-08-29 00:33:50
tags:
	- freemarker
categories:
	- bug记录
---

## 前言
在最近的工作中遇到了一个做一个导出功能时遇到了一个很奇怪的事情，逻辑是先做一个export方法上传到文件服务器上，然后重定向到一个doExport方法中，这个doExport方法中是去判断这个文件是否生成（之前生成Excel文件是异步线程生成的），如果没有生成，则转到一个export.ftl的freemarker页面，这个页面中去不断reload去调用这个doExport方法，直到导出了文件。但是在本地测试的时候，总发现doExport方法会无限的将请求再转发到export方法中，然后就一直产生了无限重定向，在浏览器中会有报无限重定向而给拦截掉。

<!-- more -->

## 现象

两个controller：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004013645.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004033317.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

模板文件：

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>报表导出</title>
</head>
<body>
<div style=" border:1px solid #ccc; margin:200px auto; height:70px;background:#eee;padding:40px;color:green;text-align:center;">
正在生成报表，请耐心等待...
</div>


<script type="text/javascript">
    setTimeout("location.reload()", 1000);
</script>、

</body>
</html>
```

本地开启debug模式进行调试：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004045615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

可以看到现在是线程nio-7001-exec-1去发送到重定向到下边的doExport方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004103884.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

此时在doExport中是新的exec-2线程去调用这个方法判断文件是否存在

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004150880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

同样的是这个线程去返回modelAndView，按理说这时候应该返回模板类freemarker中的export.ftl，但是它没有去返回freemarker文件，而是将这个请求转发到了export方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004206983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

而且是同一个线程，是容器内跳转。之后因为每次export方法都会去生成一个新的随机的文件名称，所以又会去调用doExport方法去下载新的文件名称的文件，直到有一个个文件内容比较少，异步线程在判断文件是否存在之前，文件导出，否则浏览器会直接拦截这么多次的重定向：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203004308698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

## 排查过程

### 是不是freemarker的文件路径配置有问题

既然是容器内跳转，而没有访问到真正的导出页面的文件，那是不是freemarker的文件访问路径配置有问题。看到本地环境的properties文件中关于freemarker配置如下：

```
# freemarker相关配置
spring.freemarker.enabled=true
spring.freemarker.suffix=.ftl
spring.freemarker.content-type=text/html
spring.freemarker.cache=false
spring.freemarker.charset=utf-8
spring.freemarker.check-template-location=true
spring.freemarker.template-loader-path=classpath:/templates/
```

这里去查看了模板文件确实是放在了classpath下的templates文件夹中的。所以配置和访问文件的路径没有问题。

---

### 是不是因为mapping重名，优先映射到了mapping地址

这里发现模板文件的名称和export的mapping地址是一样的，则在exprot方法上加了一个1,saleDetail/exprot1这样，然后再次debug发现，这时候不会再重新容器内跳转到这个方法，但是页面直接报了404错误，这说明还是没有去加载到模板文件。

---

### 进行一个简答的路由到模板文件的测试

这时候去写了一个helloController，然后在templetes下新建了一个hello.ftl模板文件，在controller中去return modelAndView，这时视图也是hello，发现也是404的错误。这说明和sendRedirect方法也没有关系，本身项目是不支持freemarker的。

最后发现在整个项目中，都没有freemarker的依赖和jar包。。。加入freemarker的依赖之后问题解决。

因为mapping地址和freemarker的模板文件地址相同，在你返回modelAndView的视图名称在mapping中找的到时，springmvc会容器内跳转，也不会报freemarker模板文件找不到的错误。因为其他工程也是这样去写的，所以就想当然的以为这个工程中也会有freemarker依赖。其实这个时候应该跟一下spring mvc的源码，就可以看到这时候跳转的是dispatcherServlet路由到的export方法，并没有走freemarker的模板文件。这里去记录一下。