---
title: springboot应用拦截器
copyright: true
date: 2018-09-05 00:33:15
tags:
	- 拦截器
categories:
	- spring
	- spring boot
---

## 背景

在工作中看到了不少项目用到了拦截器，这里去总结一下spring-boot使用拦截器。拦截器是Spring提供的HandlerInterceptor（拦截器），其功能和过滤器类似，但是提供更精细的控制能力：在request被响应之前、request被响应之后、视图渲染之前以及request全部结束之后。我们不能通过拦截器修改request的内容，但可以通过抛出异常（或者返回false）来暂停request的执行。
<!-- more -->

## 使用步骤

配置拦截器也很简单，spring给我们提供了WebMvcConfigurerAdapter，我们在addInterceptors方法中添加注册拦截器即可。总结起来就是三步：

1.创建我们自己的拦截器类并实现HandlerInterceptor接口。

2.创建一个Java类继承WebMvcConfigurerAdapter，并重写addInterceptors方法。

3.实例化我们自定义的拦截器，然后将对象手动添加到拦截器链中。

### 代码示例

#### 自定义Session信息写入ThreadLocal

```
@Slf4j
@Component
public class SessionInterceptor implements HandlerInterceptor {

    @Resource
    private RequestHelper requestHelper;

    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
        log.info("SessionInterceptor preHandle方法，在请求方法之前调用，Controller方法调用之前");
        // MOCK一个SessionUser对象，放入ThreadLocal中
        SessionUser sessionUser = new SessionUser();
        sessionUser.setId(2L).setName("夸克");
        requestHelper.setSessionUser(sessionUser);
        // 只有这个方法返回true 请求才能继续下去
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView)
            throws Exception {
        log.info("SessionInterceptor postHandle方法，请求处理之后调用，但是在视图被渲染之前（Controller方法调用之后）");
        // 这里可以去做sessionUser的清除 防止内存泄漏
        requestHelper.clearSession();
    }

    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
        log.info("SessionInterceptor afterCompletion方法，在整个请求结束之后调用，也就是在Dispatcher渲染了整个视图之后进行（主要进行资源清理工作）");
        if (Objects.nonNull(requestHelper.getSessionUser())) {
            requestHelper.clearSession();
        }
    }
}
```

可以看到实现了HandlerInterceptor接口之后，要实现其中的三个方法。

preHandle方法：在请求controller方法之前调用，这里就可以做一些session对象的校验及写入ThreadLocal方便方法调用等。只有这个方法返回true，请求才能继续下去。

postHandle方法：这个方法是在请求了controller方法之后但在视图渲染之前调用的。这里可以去做ThreadLocal中资源的清除。

afterCompletion方法：这个方法是在整个请求结束之后调用的，也就是在Dispatcher渲染整个视图之后进行的，主要进行资源清理工作。（这里也是去补偿了ThreadLocal中资源的清除）。

#### 注册拦截器

在WebMvcConfigurerAdapter的子类中注册这个拦截器。WebMvcConfigurerAdapter看名字就提供了很多springmvc关于web访问的配置。

```
@Component
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    /**
     * sessionInterceptor不需要拦截的请求
     * 比如swagger的请求、比如一些静态资源的访问、比如错误统一处理的页面
     */
    private static final String[] EXCLUDE_SESSION_PATH= {};

    @Resource
    private SessionInterceptor sessionInterceptor;

    @Resource
    private MyInterceptor myInterceptor;

    /**
     * 对所有的拦截器组成一个拦截器链
     * addPathPatterns 用于添加拦截规则
     * excludePathPatterns 用户排除拦截
     *
     * @param registry 拦截器注册对象
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 加入自定义拦截器到
  //      registry.addInterceptor(myInterceptor);
       registry.addInterceptor(sessionInterceptor).excludePathPatterns(EXCLUDE_SESSION_PATH);
        super.addInterceptors(registry);
    }
}

```

通过spring管理把自定义的拦截器注册成bean对象，然后通过register的addInterceptor方法注册到拦截器执行链中，这里也可以设置包括/过滤的访问地址等相关子属性。

我们写了个controller去测试这个自定义拦截器。其中helloService是把ThreadLocal中的对象给直接返回

```
@RestController
public class ThreadLocalController {


    @Resource
    private HelloService helloService;

    /**
     * 在拦截器中用ThreadLocal
     *
     * @return SessionUser
     */
    @GetMapping(value = "/threadLocal/getSessionUser")
    public SessionUser getSessionUser() {
        return helloService.getThreadSessionUser();
    }
}
```

```
 @Override
    public SessionUser getThreadSessionUser() {
        return requestHelper.getSessionUser();
    }
```

在浏览器中访问http://localhost:7001/threadLocal/getSessionUser 可以看到结果返回：

![](http://pcsb3jzo3.bkt.clouddn.com/WX20180908-173854.png)

这说明我们的拦截器是正确将sessionUser设置进入ThreadLocal对象的。看控制台的日志输入：

![](http://pcsb3jzo3.bkt.clouddn.com/WX20180908-174122.png)

#### 多个自定义拦截器的执行顺序

如果再定义一个自定义拦截器，那么执行的顺序是什么呢？

```
@Slf4j
@Component
public class MyInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest,
            HttpServletResponse httpServletResponse, Object o) throws Exception {
        log.info("MyInterceptor preHandle方法，在请求方法之前调用，Controller方法调用之前");
        // 返回true才能继续执行
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest httpServletRequest,
            HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView)
            throws Exception {
        log.info("MyInterceptor postHandler方法，请求处理之后调用，但是在视图被渲染之前（Controller方法调用之后）");

    }

    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest,
            HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
        log.info("MyInterceptor afterCompletion方法，在整个请求结束之后调用，也就是在Dispatcher渲染了整个视图之后进行"
                + "主要进行资源清理工作");
    }
}
```

在webMvcConfig中注册这两个拦截器，并且在浏览器中访问同样的url。看到控制台的输入。

![](http://pcsb3jzo3.bkt.clouddn.com/WX20180908-174857.png)

在webMvcConfig中，我们注册的顺序是先注册了myInterceptor，然后注册了SessionInterceptor，看到的执行顺序为：

preHandle方法：myInterceptor先执行

postHandle方法：sessionInterceptor先执行

afterCompletion方法：sessionInterceptor先执行。

可见多个自定义拦截器在执行链中的执行顺序是与注册顺序相关的，preHandle方法是先注册先执行，其他两个方法是后注册的先执行。具体执行的顺序的分析可以见下图。

![](http://pcsb3jzo3.bkt.clouddn.com/20180603133007923.png)

### 拦截器的缺点

它依赖于web框架，在SpringMVC中就是依赖于SpringMVC框架。在实现上,基于Java的反射机制，属于面向切面编程（AOP）的一种运用，**就是在service或者一个方法前，调用一个方法，或者在方法后，调用一个方法，比如动态代理就是拦截器的简单实现，在调用方法前打印出字符串（或者做其它业务逻辑的操作），也可以在调用方法后打印出字符串，甚至在抛出异常的时候做业务逻辑的操作。**由于拦截器是基于web框架的调用，因此可以使用Spring的依赖注入（DI）进行一些业务操作，同时一个拦截器实例在一个controller生命周期之内可以多次调用。**但是缺点是只能对controller请求进行拦截**，对其他的一些比如直接访问静态资源的请求则没办法进行拦截处理。 

同时，反射也有可能对性能有些影响。

### HandlerInterceptor接口分析

```
public interface HandlerInterceptor {
 
	
	 /** 
     * preHandle方法是进行处理器拦截用的，顾名思义，该方法将在Controller处理之前进行调用，SpringMVC中的Interceptor拦截器是链式的，可以同时存在 
     * 多个Interceptor，然后SpringMVC会根据声明的前后顺序一个接一个的执行，而且所有的Interceptor中的preHandle方法都会在 
     * Controller方法调用之前调用。SpringMVC的这种Interceptor链式结构也是可以进行中断的，这种中断方式是令preHandle的返 
     * 回值为false，当preHandle的返回值为false的时候整个请求就结束了。 
     */  
	boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
	    throws Exception;
 
	
	/** 
     * 这个方法只会在当前这个Interceptor的preHandle方法返回值为true的时候才会执行。postHandle是进行处理器拦截用的，它的执行时间是在处理器进行处理之 
     * 后，也就是在Controller的方法调用之后执行，但是它会在DispatcherServlet进行视图的渲染之前执行，也就是说在这个方法中你可以对ModelAndView进行操 
     * 作。这个方法的链式结构跟正常访问的方向是相反的，也就是说先声明的Interceptor拦截器该方法反而会后调用，这跟Struts2里面的拦截器的执行过程有点像， 
     * 只是Struts2里面的intercept方法中要手动的调用ActionInvocation的invoke方法，Struts2中调用ActionInvocation的invoke方法就是调用下一个Interceptor 
     * 或者是调用action，然后要在Interceptor之前调用的内容都写在调用invoke之前，要在Interceptor之后调用的内容都写在调用invoke方法之后。 
     */
	void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception;
 
	
	/** 
     * 该方法也是需要当前对应的Interceptor的preHandle方法的返回值为true时才会执行。该方法将在整个请求完成之后，也就是DispatcherServlet渲染了视图执行， 
     * 这个方法的主要作用是用于清理资源的，当然这个方法也只能在当前这个Interceptor的preHandle方法的返回值为true时才会执行。 
     */ 
	void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception;
 
}
```

#### HandlerInterceptorAdapter

springMvc还提供了HandlerInterceptorAdapter这个抽象类，这个抽象类中实现了AsyncHandlerInterceptor接口，而AsyncHandlerInterceptor接口又继承了HandlerInterceptor接口，我们可以首先看下AsyncHandlerInterceptor接口：

```
public interface AsyncHandlerInterceptor extends HandlerInterceptor {

	/**
	 * Called instead of {@code postHandle} and {@code afterCompletion}, when
	 * the a handler is being executed concurrently.
	 * <p>Implementations may use the provided request and response but should
	 * avoid modifying them in ways that would conflict with the concurrent
	 * execution of the handler. A typical use of this method would be to
	 * clean up thread-local variables.
	 * @param request the current request
	 * @param response the current response
	 * @param handler the handler (or {@link HandlerMethod}) that started async
	 * execution, for type and/or instance examination
	 * @throws Exception in case of errors
	 */
	void afterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception;

}
```

可以看到在这个接口中添加了一个afterConcurrentHandlingStarted方法。 该方法是用来处理异步请求。当Controller中有异步请求方法的时候会触发该方法。异步请求先支持preHandle、然后执行afterConcurrentHandlingStarted，之后才会执行postHandle的方法。

比如现在我们配置了一个拦截器是用来拦截异步请求的：

```
@Component
@Slf4j
public class MyAsyncHandlerInterceptor extends HandlerInterceptorAdapter {

    /**
     * 该方法是用来处理异步请求。当Controller中有异步请求方法的时候会触发该方法。
     * 异步请求先支持preHandle、然后执行afterConcurrentHandlingStarted。
     *
     * @param request
     * @param response
     * @param handler
     * @throws Exception
     */
    @Override
    public void afterConcurrentHandlingStarted(HttpServletRequest request,
            HttpServletResponse response, Object handler) throws Exception {
        super.afterConcurrentHandlingStarted(request, response, handler);
        log.info("MyAsyncHandlerInterceptor afterConcurrentHandlingStarted 方法执行");
    }

}
```

这个时候我们有个异步请求处理：(对于springMvc的异步请求可以看看这篇博客：[springmvc的异步请求](https://blog.csdn.net/u012410733/article/details/52124333))

```
@RestController
@Slf4j
public class HelloController {

    @GetMapping(value = "/hello")
    public Callable<String> sayHello() {
      return () -> "controller";
    }
}
```

这时在浏览器中调用我们的异步请求，可以看到控制台中输出

![](http://pcsb3jzo3.bkt.clouddn.com/WX20180908-192730.png)

可见这时先调用的是afterConcurrentHandlingStarted方法，而后调用的是postHandle方法。

我们再去看适配器HandlerInterceptorAdapter的代码：

```
public abstract class HandlerInterceptorAdapter implements AsyncHandlerInterceptor {

	/**
	 * This implementation always returns {@code true}.
	 */
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return true;
	}

	/**
	 * This implementation is empty.
	 */
	@Override
	public void postHandle(
			HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception {
	}

	/**
	 * This implementation is empty.
	 */
	@Override
	public void afterCompletion(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
	}

	/**
	 * This implementation is empty.
	 */
	@Override
	public void afterConcurrentHandlingStarted(
			HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
	}

}
```

可以看到preHandle方法默认实现返回了true，比如我们只想去定义一个拦截器去在方法执行完之后去释放掉一些资源，如果去实现HandlerInterceptor则显得有点麻烦。这里只要去继承这个抽象类，实现afterCompletion方法即可。

### github

上述代码都能在gitHub上看到：
https://github.com/zhanglijun1217/spring-boot-demo