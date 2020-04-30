---
title: 如何让PropertySource支持yml配置文件
tags:
  - SPRING BOOT
  - JAVA
  - 配置文件
  - YML
date: 2019-12-27 11:56:19
category: CODING
---

现在越来越多的系统使用yaml文件配置了，因为有层次感，而且字少，什么东西都很容易看清楚。现在spring boot也支持application.yml作为默认的配置文件了。

然而如果我们想用自己的yml配置文件该怎么做呢？

一般的properties文件自然而然想到用[PropertySource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/PropertySource.html)这个来引入，一般是这样的：

```java
@PropertySource("classpath:target.yml")
```

不过你要是真这么做的话所有拿到的数据都是null。
__而且没有报错__

既然spring都默认支持怎么读yaml配置了，为什么这个不行呢？

仔细看PropertySource的代码，发现一个叫factory的属性，这个东西是让用户自己来配置PropertySourceFactory。这个东西是用来将文件读取出来原来的resource解析成PropertyResource的。

接下来问题就简单了，既然spring boot能自动解析application.yml文件，那么一定有对应的YamlPropertySourceFactory吧！

然而并没有。

非常奇怪的是spring boot自带的[YamlPropertySourceLoader](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/env/YamlPropertySourceLoader.html)，却没有做最后一步弄个PropertySourceFactory包起来。

没办法，只能自己写一个了

```java
public class YamlPropertySourceFactory implements PropertySourceFactory {

    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
        YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
        factory.setResources(resource.getResource());
        factory.afterPropertiesSet();
        Properties ymlProperties = factory.getObject();
        String propertyName = name != null ? name : resource.getResource().getFilename();
        return new PropertiesPropertySource(propertyName, ymlProperties);
    }
}
```

之后重新配置一下PropertySource

```java
@PropertySource(value = "classpath:target.yml", factory = YamlPropertySourceFactory.class)
```

搞定。
