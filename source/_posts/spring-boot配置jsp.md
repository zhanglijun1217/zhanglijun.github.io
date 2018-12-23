---
title: spring boot配置jsp
copyright: true
date: 2018-10-23 00:05:33
tags:
	- spring boot
	- actuator
categories:
	- spring
	- spring boot
---

## spring-boot中jsp的使用

jsp是之前在学习java开发中会学习到的知识，虽然现在公司中虽然使用jsp越来越少，但是spring-boot配置jsp的使用还是应该去记录一下。

<!-- more -->

### 相关依赖增加

这里要加入一些依赖：

```
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- jsper渲染引擎 -->
      <dependency>
          <groupId>org.apache.tomcat.embed</groupId>
          <artifactId>tomcat-embed-jasper</artifactId>
      </dependency>
      <!-- 内置tomact -->
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-tomcat</artifactId>
      </dependency>

      <!-- jstl 依赖 -->
      <dependency>
          <groupId>javax.servlet</groupId>
          <artifactId>jstl</artifactId>
      </dependency>
```

还要需要注意的是spring-boot默认打包方式jar包的形式，这里要换成war包的方式。

### 激活传统Servlet web部署

springboot1.4版本之后通过实现org.springframework.boot.web.support.SpringBootServletInitializer抽象类中的抽象方法来将启动类添加到souce中

```java
public class JspConfig extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        // source启动类 告知一些静态资源
        builder.sources(SpringBootDemoApplication.class);
        return builder;
    }
}
```

### 加入资源目录位置

在项目的src/main目录下建立一个webapp文件夹，这个webapp目录下建立WEB-INFO和jsp文件夹，写一个index.jsp文件作为之后的测试页面。

目录：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203001445552.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

```jsp
<html>
<body>
hello, ${message}
</body>
</html>
```

### 设置访问资源文件的前缀和后缀

在application.properties配置文件中配置访问jsp文件中的prefix和suffix，注意这里这prefix中的开头和结尾的/是不能省略的，否则会访问不到你的资源。

```
# 访问jsp资源的前缀和后缀
spring.mvc.view.prefix = /WEB-INFO/jsp/
spring.mvc.view.suffix = .jsp
```

### 写一个test的controller

在配置好了之后，写一个controller作为入口去访问这个jsp文件

```java
@Controller
public class JspController {


    @RequestMapping(value = "/index")
    public String index(Model model) {
        model.addAttribute("message", "zlj");

        return "index";
    }
}
```

这时候在浏览器中输入localhost:7001/index即可访问到我们返回给index.jsp中message占位符的字符串值。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203001432718.png)

