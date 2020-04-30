---
title: Sprint Boot @Value 注入无效的情况
date: 2018-01-22 13:48:34
tags: 
- JAVA
- SPRING
- SPRING BOOT
category: CODING
---

```java
@Service
public class SomeService {
    @Value("${a}")
    String a;
    @Value("${b}")
    String b;
    
    private C c = new C(a, b);
}
```

上面的代码有什么问题吗？有！
这个时候new C(a, b)里面的a和b并不是上面被@Value注入的值，而是null。这是为什么呢？

首先我们看一下Spring在初始化Bean的过程：
![spring_context_callback.png](http://ww1.sinaimg.cn/large/718163e1gy1g8ncpdmivjj20zf02raa1.jpg)

解释一下：
1. 读取定义
2. Bean Factory Post Processing
3. 实例化
4. 属性注入
5. Bean Post Processing

这个问题的原因就很明显了。@Value注入是在第四步，而初始化变量是在第三部，这个使用变量还没有被注入，自然为null了。

解决的方法也很简单，把需要初始化的变量放到第四步去实例化就可以了。下面给出一个解决方法：
```java
@Service
public class SomeService {
    private C c
    @Autowired
    public void setC(@Value("${a}") String a,
                     @Value("${b}") String b) {
        c = new C(a, b);
    }
}
```