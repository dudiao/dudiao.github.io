---
title: springboot兼容返回json和xml
date: 2019-01-18 17:00:54
tags:
  - springboot
  - json
  - xml
---

### 1. 返回json

springMVC中，默认返回的json格式
```java
@RestController
public class MyController {

    @RequestMapping("/thing")
    public MyThing thing() {
        return ObjectRestResponse.ok("F_RA1");
    }
    
}
```

访问`localhost:8080/thing`，默认返回的是json

### 2. 返回xml

如果想返回xml格式，需要加入依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

> springboot项目，相关版本统一由springboot parent管理，所以不用指定版本号

访问`localhost:8080/thing`，同时，请求头加入：`Accept: text/xml`，返回结果示例如下：

```xml
<ObjectRestResponse>
    <status>200</status>
    <data>F_RA1</data>
</ObjectRestResponse>
```