---
title: SpringBoot自动装配之坑
date: 2022-07-04 22:20:33
categories:
  - SpringBoot
tags:
  - SpringBoot
---

# SpringBoot自动装配之坑

今天在工作中遇到个自动装配的问题，简要记录，避免再次入坑。

代码结构如下：

![image-20220704222749198](https://raw.githubusercontent.com/zhenglinsmile/picture/master/img/image-20220704222749198.png)

直接上代码：

### 正常情况

```java
@Slf4j
@Data
public class FirstBean {

  public FirstBean() {
      log.info("Init FirstBean");
  }

}

@Configuration
public class FirstConfiguration {

  @Bean
  public FirstBean firstBean() {
    return new FirstBean();
  }

}


@Slf4j
@Data
public class SecondBean {

  public SecondBean() {
    log.info("Init SecondBean");
  }
}

@Configuration
public class SecondConfiguration {

  @Bean
  public SecondBean secondBean() {
    return new SecondBean();
  }

}

// Init FirstBean
// Init SecondBean
```

### 异常情况

```java
@Configuration
public class FirstConfiguration {

  @Bean
  // 增加@ConditionalOnBean(value = SecondBean.class)
  @ConditionalOnBean(value = SecondBean.class)
  public FirstBean firstBean() {
    return new FirstBean();
  }

}
// Init SecondBean
// 未能正常加载 FirstBean
```

### 原因

我们查看`ConditionalOnBean`源码，注释中最后一句描述：**如果一个目标类的注入依赖另一个自动装配类，需要确保它们的加载顺序。**

所以，上面的例子中，问题根源在于加载`FirstBean`时`SecondBean`还未被加载，所以条件不成立，注入失败。

```java
/**
* The condition can only match the bean definitions that have been processed by the application context so far and, as such, it is strongly recommended to use this condition on auto-configuration classes only. If a candidate bean may be created by another auto-configuration, make sure that the one using this condition runs after.
*/
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {}
```

### 解决

这个问题的核心在于让其正确的加载，SpringBoot有以下注解可以处理**多个自动装配类加载顺序**：`@AutoConfigureAfter`、`@AutoConfigureBefore`、`@AutoConfigureOrder`

```java
@Configuration
@AutoConfigureAfter(value = SecondConfiguration.class)
public class FirstConfiguration {

  @Bean
  @ConditionalOnBean(value = SecondBean.class)
  public FirstBean firstBean() {
    return new FirstBean();
  }

}
```

实测即使加入如上的配置`@AutoConfigureAfter(value = SecondConfiguration.class)`，依然未能正确加载，还需要在项目资源目录下增加`META-INF/spring.factories`文件，内容如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  site.zlin.bootstrap.SecondConfiguration,\
  site.zlin.bootstrap.FirstConfiguration
```

这样问题就得以修复。

但是，在生产中我遇到另一种情况，自动装配文件中的内容如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  site.zlin.bootstrap.BootStrap
```

**BootStrap 源码**

```java
// 指定包扫描路径
@ComponentScan(value = "site.zlin.bootstrap")
public class BootStrap {

}
```

这种情况下，我们将配置文件内容调整为如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  site.zlin.bootstrap.SecondConfiguration,\
  site.zlin.bootstrap.FirstConfiguration,\
  site.zlin.bootstrap.BootStrap
```

经过测试，`FirstBean`依然未能被正确注入，经过查找资料分析，是因为包中的类被扫描了多次，具有不确定性。调整如下：

```java
@ComponentScan(value = "site.zlin.bootstrap", excludeFilters = {
    @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = {FirstConfiguration.class,
        SecondConfiguration.class})})
public class BootStrap {

}
```

再次测试，终于OK。