---
title: spring中根据Application获取BEAN的工具类
copyright: true
date: 2018-08-12 22:16:44
tags:
    - spring
categories:
	- spring
	- 框架应用
---

## 背景
- 在最近的开发工作中，用到了策略模式（之前也写过关于策略模式这个设计模式的学习，但是之前那个不是在spring框架中），这时候策略中的context或者factory就要去动态的根据调用的策略类型不同去拿到对应的bean对象，这里去了解了一个通过application context拿取bean的工具类，这里记录一下。
<!--more-->
- 话不多说，直接上代码
```
@Component
@Slf4j
public class ApplicationContextBeanUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    /**
     * 利用aware注入application
     * @param applicationContext
     * @throws BeansException
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        // 注入application
            ApplicationContextBeanUtil.applicationContext = applicationContext;
    }

    private static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    /**
     * 通过name获取bean
     * @param name
     * @return
     */
    public static Object getBean(String name) {
        return getApplicationContext().getBean(name);
    }

    /**
     * 通过class获取bean
     * @param clazz
     * @param <T>
     * @return
     */
    public static <T> T getBean(Class<T> clazz) {
        return getApplicationContext().getBean(clazz);
    }

    /**
     * 通过name和class获取bean
     * @param name
     * @param clazz
     * @param <T>
     * @return
     */
    public static <T> T getBean(String name, Class<T> clazz) {
        return getApplicationContext().getBean(name, clazz);
    }

    /**
     * 根据clazz类型获取spring容器中的对象
     * @param clazz
     * @param <T>
     * @return
     */
    public static <T> Map<String, T> getBeansOfType(Class<T> clazz) {
        return getApplicationContext().getBeansOfType(clazz);
    }

    /**
     * 根据注解类从容器中获取对象
     * @param clazz
     * @param <T>
     * @return
     */
    public static <T> Map<String, Object> getBeansOfAnnotation(Class<? extends Annotation> clazz) {
        return getApplicationContext().getBeansWithAnnotation(clazz);
    }
}

```

- 这是通过实现ApplicationContextAware接口去实现注入application的，这里应该注意几点：
1. application应该是静态的。这个Util类应该是在别的类中直接调用获取bean的静态方法，所以注入的applicationContext应该都是该类的静态变量。
2. 要用注解或者在xml文件中将这个Util配置成bean。（这里用的spring boot，就直接配置的扫描）。
3. 在其中提供了一些获取bean的方法。
- 这里去记录下，方便在之后的工作中遇到了之后去直接使用