---
title: Spring Resource
date: 2021-07-27 15:22:21
categories:
  - Spring
tags:
  - Spring
---

# Spring Resource 抽象

- ResourceLoader

```java
// ApplicationContext 实际上也是一个ResourceLoader

@Component
public class ResourceLoaderContext implements ResourceLoaderAware {
    private ResourceLoader resourceLoader;
    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }
    public ResourceLoader getResourceLoader() {
        return resourceLoader;
    }
}

@Autowired
private ResourceLoaderContext resourceLoaderContext;

@Test
public void resourceLoaderTest() throws IOException {
  // 这里会自动根据路径选择不同的资源加载器
    final Resource resource = resourceLoaderContext.getResourceLoader().getResource("file:/Users/zhenglin/learn-space/RuoYun/framework/src/main/java/zlin/site/framework/Foo.java");
    try (final InputStream stream = resource.getInputStream()) {
        final String content = IOUtils.toString(stream);
        log.info(content);
    }
}
```

- ResourcePatternResolver
- Resource 注入

```java
// 单个资源文件注入
@Component
@Getter
public class ResourcePathBean {
    private final Resource template;
    public ResourcePathBean(@Value("${template.path}") Resource template) {
        this.template = template;
    }
}

// 多个资源文件注入
@ConfigurationProperties(prefix = "templates")
@Component
@Getter
public class MultipleResourcePathBean {
    private final List<Resource> templates;
    public MultipleResourcePathBean(@Value("${path}") List<Resource> templates) {
        this.templates = templates;
    }
}
```
