---
layout: post
title:  "Spring Boot logback 配置"
date:   2019-07-04 14:06:05
categories: JAVA logback
excerpt: spring boot logback 详解
---

* content
{:toc}

## spring boot logback xml
按照推荐命名日志文件 logback-spring.xml

---

## 集成配置

```
logging:
  config: classpath:logback-spring.xml
  level:
    dao: debug
    org:
      mybatis: debug
```

---

### 下面是具体logback配置

```


```