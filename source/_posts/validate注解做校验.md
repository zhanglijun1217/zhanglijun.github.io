---
title: validate注解做校验
copyright: true
date: 2018-08-18 13:25:26
tags:
    - validate注解
categories:
    - spring
    - 注解
---
## 前言
之前也用过hibernate的校验注解，但是没有去做一个总结，这里参考一篇博客去做一个总结。简述JSR303/JSR-349，hibernate validation，spring validation之间的关系。JSR303是一项标准,JSR-349是其的升级版本，添加了一些新特性，他们规定一些校验规范即校验注解，如@Null，@NotNull，@Pattern，他们位于javax.validation.constraints包下，只提供规范不提供实现。而hibernate validation是对这个规范的实践（不要将hibernate和数据库orm框架联系在一起），他提供了相应的实现，并增加了一些其他校验注解，如@Email，@Length，@Range等等，他们位于org.hibernate.validator.constraints包下。而万能的spring为了给开发者提供便捷，对hibernate validation进行了二次封装，显示校验validated bean时，你可以使用spring validation或者hibernate validation，而spring validation另一个特性，便是其在springmvc模块中添加了自动校验，并将校验信息封装进了特定的类中。这无疑便捷了我们的web开发。本文主要介绍在springmvc中自动校验的机制。
<!-- more -->
## 引入依赖
因为我们构建的是spring boot项目，所以直接引入web的starter的依赖即可。
```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

如果查看其子依赖，可以发现如下的依赖：
```
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

## 进行校验
### 校验的实体类
这里用了lombok的@Data注解，也是非常推荐大家使用的一个插件。
```
@Data
public class ValidateBO {

    @NotBlank(message = "name不能为空")
    private String name;

    @Min(value = 18, message = "年龄不能小于18岁")
    private Integer age;

    @Email(message = "email格式错误")
    private String email;

    /**
     * 自定义注解 不能包含空格字符串
     */
    @CannotHaveBlank
    private String blank;

    /**
     * 正则校验
     */
    @Pattern(regexp = "^1(3|4|5|7|8)\\d{9}$", message = "手机号码格式错误")
    private String phone;

}

```
可以看到这里用到了一些常见的注解。（自定义校验注解在下边会提到）
### controller进行校验
在controller中进行这个字段的校验，可以看到每个需要校验的对象，都需要一个BindingResult去承接校验的结果，并且也要对要校验的类去加上@Validated注解。
```
 @GetMapping(value = "/validate")
    public String validate(ValidateBO validateBO, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            StringBuilder stringBuilder = new StringBuilder();
            for (String s : bindingResult.getFieldErrors().stream()
                    .map(FieldError::getDefaultMessage).collect(Collectors.toList())) {
                stringBuilder.append(s);
            }
            return stringBuilder.toString();
        }
        return "success";
    }
```
启动项目，在url中输入对项目进行debug。
![debug这次请求](http://pcsb3jzo3.bkt.clouddn.com/WX20180819-173447@2x.png)
可以看到，校验在碰到第一个字段不符合要求之后，并不是直接返回错误，而是会对所有的要校验字段去校验。当然这个也是可以配置的，下边会提到fast-fail的配置。最后返回的结果：
![](http://pcsb3jzo3.bkt.clouddn.com/WX20180819-173919.png)
这里是打印出了这个对象中所有的错误。
### 常见的校验注解
JSR提供的校验注解:
```
@Null   被注释的元素必须为 null    
@NotNull    被注释的元素必须不为 null    
@AssertTrue     被注释的元素必须为 true    
@AssertFalse    被注释的元素必须为 false    
@Min(value)     被注释的元素必须是一个数字，其值必须大于等于指定的最小值    
@Max(value)     被注释的元素必须是一个数字，其值必须小于等于指定的最大值    
@DecimalMin(value)  被注释的元素必须是一个数字，其值必须大于等于指定的最小值    
@DecimalMax(value)  被注释的元素必须是一个数字，其值必须小于等于指定的最大值    
@Size(max=, min=)   被注释的元素的大小必须在指定的范围内    
@Digits (integer, fraction)     被注释的元素必须是一个数字，其值必须在可接受的范围内    
@Past   被注释的元素必须是一个过去的日期    
@Future     被注释的元素必须是一个将来的日期    
@Pattern(regex=,flag=)  被注释的元素必须符合指定的正则表达式
```
Hibernate Validator提供的校验注解：
```
@NotBlank(message =)   验证字符串非null，且长度必须大于0    
@Email  被注释的元素必须是电子邮箱地址    
@Length(min=,max=)  被注释的字符串的大小必须在指定的范围内    
@NotEmpty   被注释的字符串的必须非空    
@Range(min=,max=,message=)  被注释的元素必须在合适的范围内
```
## 分组校验
### 场景
如果同一个类，在不同的使用场景下有不同的校验规则，那么可以使用分组校验。未成年人是不能喝酒的，而在其他场景下我们不做特殊的限制，这个需求如何体现同一个实体，不同的校验规则呢？
### 校验对象
```
@Data
public class ValidateByGroupBO {

    /**
     * 只有adult组内才进行 validate 校验
     */
    @Min(value = 18, groups = {Adult.class})
    private Integer age;

    public interface Adult{}

    public interface Minor{}
}
```
这就定义了只有在在adult组内才会进行最小值18的校验。
### 进行验证
```
 /**
     * 喝酒这个去校验了年龄值，因为只有adult这个组才去校验年龄
     * @param validateByGroupBO
     * @param bindingResult
     * @return
     */
    @GetMapping(value = "/drink")
    public String drink(@Validated({ValidateByGroupBO.Adult.class}) ValidateByGroupBO validateByGroupBO, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            // 处理错误
            return "false";
        }
        return "success";
    }

    /**
     * 生活不需要去校验adult的分组 就不去校验对应的age的最小值
     * @param validateByGroupBO
     * @param bindingResult
     * @return
     */
    @GetMapping(value = "live")
    public String live(@Validated ValidateByGroupBO validateByGroupBO, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            // 错误处理
            return "false";
        }
        return "success";
    }
```
运行之后，在url中输入http://localhost:7001/drink?age=10，第一个返回false，因为指定了使用其中的Adult分组，会开启对age的校验。输入http://localhost:7001/live?age=10则会不去校验age的大小，返回success。喝酒要校验是否成年，而生活不用，类似的场景还是很容易碰到的。
## 自定义注解
### 实现一个注解
这里去实现一个字符串中不能含有blank空格。主要分为两步：
- 先去定义这个注解，其中validatedBy指定的是真正去做校验的实体类。而其中的groups和payload可以直接用默认。
```
@Target({METHOD, FIELD, ANNOTATION_TYPE, ElementType.CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
// 这个注解是引入真正的去做验证的类
@Constraint(validatedBy = {CannotHaveBlankValidator.class})
public @interface CannotHaveBlank {

    // 默认错误信息
    String message() default "不能包含空格";

    // 分组
    Class<?>[] groups() default {};

    //负载
    Class<? extends Payload>[] payload() default {};

    //指定多个时使用
    @Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE})
    @Retention(RUNTIME)
    @Documented
    @interface List {
        CannotHaveBlank[] value();
    }
}

```
- 第二步是去实现真正去做校验的实体类
```
public class CannotHaveBlankValidator implements ConstraintValidator<CannotHaveBlank, String> {


    @Override
    public void initialize(CannotHaveBlank cannotHaveBlank) {

    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        //null时不进行校验
        if (value != null && value.contains(" ")) {
            //获取默认提示信息
            String defaultConstraintMessageTemplate = context.getDefaultConstraintMessageTemplate();
            System.out.println("default message :" + defaultConstraintMessageTemplate);
            //禁用默认提示信息
            context.disableDefaultConstraintViolation();
            //设置提示语
            context.buildConstraintViolationWithTemplate("can not contains blank")
                    .addConstraintViolation();
            return false;
        }
        return true;
    }
}
```
这里去实现类去实现了ConstraintValidator接口，这个接口中包含一个初始化事件方法和一个判断是否合法的方法：
```
package javax.validation;

import java.lang.annotation.Annotation;

public interface ConstraintValidator<A extends Annotation, T> {
    void initialize(A var1);

    boolean isValid(T var1, ConstraintValidatorContext var2);
}

```
其中的A泛型参数是上一步定义的注解类，泛型T是要去校验的字段类型。
ConstraintValidatorContext 这个参数上下文包含了认证中所有的信息，我们可以利用这个上下文实现获取默认错误提示信息，禁用错误提示信息，改写错误提示信息等操作。

### 自定义注解进行校验
还是用第一个controller去验证这个自定义注解。因为要校验的对象中加入了自定义注解的blank字段。启动项目，输入http://localhost:7001/validate?blank=19 209（这里加了空格），可以看到返回值是：
![](http://pcsb3jzo3.bkt.clouddn.com/WX20180819-180207.png)
说明自定义注解起到了作用。

## @Valid 和 @Validated的区别
https://blog.csdn.net/qq_27680317/article/details/79970590
这篇讲的很清晰了。

## aop
很显然，如果我们每个controller中的方法都去写BindingResult就显得很麻烦了，其实我们就是要对参数进行校验并且输出到log中，这就很自然的想到了aop。

### 注解标识
定义一个注解去标识使用了hibernate validate注解
```
package com.zhanglijun.springbootdemo.domain.anno;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import javax.validation.groups.Default;

/**
 * 用来表示开启hibernate校验的注解
 * @author 夸克
 * @create 2018/8/19 22:28
 */
@Target({ElementType.METHOD,ElementType.TYPE,ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableValidate {

    Class<?> [] groups() default { Default.class };//校验分组信息
}

```

### 定义切面
```
package com.zhanglijun.springbootdemo.aspect;


import com.zhanglijun.springbootdemo.domain.anno.EnableValidate;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.HashSet;
import java.util.Set;
import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.executable.ExecutableValidator;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.hibernate.validator.HibernateValidator;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.stereotype.Component;
import org.springframework.util.ClassUtils;

/**
 * @author 夸克
 * @create 2018/8/19 18:45
 */
@Slf4j
@Aspect
@Component
public class ValidatorAspect {

    /**
     * 获取校验的工厂的信息
     */
    private static final Validator validator = Validation.byProvider(HibernateValidator.class)
            .configure()
            //快速失败模式开启，当检测到有一项失败立即停止
            .failFast(true)
            .buildValidatorFactory().getValidator();

    /**
     * point配置
     */
    @Pointcut("execution(* com.zhanglijun.springbootdemo.web.controller..*.*(..))")
    public void pointcut() {
    }


    /**
     * @desction: 校验步骤 1.首先校验是否含有基本的Hibernate validator 注解，有异常抛出 2.校验方法参数中是否含有EgValidate注解，获取分组信息，进行Bean级别的校验，有异常抛出
     * 3.查看当前的方法中（优先级高）(或者父类、父接口)是否含有EgValidate注解，没有获取当前类的中是否是否含有EgValidate注解，获取分组信息，针对每一个非基本类型Bean进行校验，有异常掏出
     * @author: wangji
     * @date: 2018/3/13 10:16
     */
    @Before("pointcut()")
    public void before(JoinPoint point) {

        //  获得切入目标对象
        Object target = point.getThis();
        // 获得切入方法参数
        Object[] args = point.getArgs();
        // 获得切入的方法
        Method method = ((MethodSignature) point.getSignature()).getMethod();
        Annotation[] classAnnotations = target.getClass().getAnnotations();
        Annotation[] methodAnnotations = method.getAnnotations();
        Annotation[][] parameterAnnotations = method.getParameterAnnotations();
        //如果方法参数有基本的注解，就进行Hibernate validator 基本的参数校验
        if (parameterAnnotations != null) {
            validMethodParams(target, method, args);
        }

        // 判断参数中是否含有EgValidate注解，进行特殊分组，Bean级别的参数校验
        int i = 0;

        //排查掉已经在参数中校验过的参数不适用类或者方法上的校验参数在次进行校验
        Set<Integer> idSet = new HashSet<>(3);
        for (Object arg : args) {
            if (arg != null) {
                if (parameterAnnotations != null) {
                    for (Annotation parameterAnnotation : parameterAnnotations[i]) {
                        if (parameterAnnotation instanceof EnableValidate) {
                            if (!ClassUtils.isPrimitiveOrWrapper(arg.getClass())) {
                                validBeanParam(arg,
                                        ((EnableValidate) parameterAnnotation).groups());
                                idSet.add(i);
                            }
                        }
                    }
                }
                i++;
            }
        }
        // 如果没有异常继续校验当前的每一个非基本类型的参数
        EnableValidate egValidate = null;
        //方法上是否有校验参数
        if (methodAnnotations != null) {
            egValidate = AnnotationUtils.findAnnotation(method, EnableValidate.class);
        }
        // 类上是否含有
        if (egValidate == null && classAnnotations != null) {
            egValidate = AnnotationUtils.findAnnotation(target.getClass(), EnableValidate.class);
        }
        // 如果在类或者方法上加了验证注解 ，则对所有非基本类型的参数对象进行验证,不管参数对象有没有加注解，使用方法上的分组
        if (egValidate != null && args != null && args.length > 0) {
            i = 0;
            for (Object arg : args) {
                if (arg != null && !ClassUtils.isPrimitiveOrWrapper(arg.getClass()) && !idSet
                        .contains(i)) {
                    validBeanParam(arg, egValidate.groups());
                }
                i++;
            }
        }

    }


    /**
     * @param obj 参数中的Bean类型参数
     * @param groups 分组信息
     * @desction: 进行参数中的Bean校验
     * @author: wangji
     * @date: 2018/3/13 10:10
     */
    private void validBeanParam(Object obj, Class<?>... groups) {
        Set<ConstraintViolation<Object>> validResult = validator.validate(obj, groups);
        throwConstraintViolationException(validResult);
    }


    /**
     * @param obj 当前的实例
     * @param method 实例的方法
     * @param params 参数
     * @desction: 对于Hibernate 基本校验Bean放在参数中的情况的校验 【例如 User getUserInfoById(@NotNull(message =
     * "不能为空") Integer id);】
     * @author: wangji
     * @date: 2018/3/13 10:11
     */
    private void validMethodParams(Object obj, Method method, Object[] params) {
        ExecutableValidator validatorParam = validator.forExecutables();
        Set<ConstraintViolation<Object>> validResult = validatorParam
                .validateParameters(obj, method, params);
        throwConstraintViolationException(validResult);
    }

    /**
     * @desction: 判断校验的结果是否存在异常
     * @author: wangji
     * @date: 2018/3/13 10:09
     */
    private void throwConstraintViolationException(Set<ConstraintViolation<Object>> validResult) {
        if (!validResult.isEmpty()) {
            throw new ConstraintViolationException(validResult.iterator().next().getMessage(),
                    validResult);
        }
    }


}

```
这个切面说的也很清除，对多处使用这个的地方都去做了一个校验。

## github
上述代码都在我的github，可以在review一下代码。
[validated注解相关](https://github.com/zhanglijun1217/spring-boot-demo/tree/master/src/main/java/com/zhanglijun/springbootdemo)

## 引用
参考的博客：
- https://www.cnkirito.moe/spring-validation/
- https://blog.csdn.net/oKuZuoZhou/article/details/81024795
- https://blog.csdn.net/u012881904/article/details/79538895
