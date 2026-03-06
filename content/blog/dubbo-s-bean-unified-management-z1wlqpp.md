---
title: Dubbo 的 Bean 统一管理
slug: dubbo-s-bean-unified-management-z1wlqpp
url: /post/dubbo-s-bean-unified-management-z1wlqpp.html
date: '2026-02-24 15:29:46+08:00'
lastmod: '2026-03-06 19:35:58+08:00'
toc: true
isCJKLanguage: true
---



# Dubbo 的 Bean 统一管理

调用外部服务，如果存在多 Bean 重复写代码且这种方式无法 @MockBean

```java
@DubboReference(timeout = 1000,version = "1.0.0")
private CollectionFacadeService collectionFacadeService;
```

改用以下方式，就可以使用 @AutoWired 进行自动注入了

```java
package *.pay.infrastructure;

import *.api.collection.service.CollectionFacadeService;
import org.apache.dubbo.config.annotation.DubboReference;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class PayDubboConfiguration {

    @DubboReference(timeout = 1000,version = "1.0.0")
    private CollectionFacadeService collectionFacadeService;

    @Bean
    @ConditionalOnMissingBean(name = "collectionFacadeService")
    public CollectionFacadeService collectionFacadeService() {
        return collectionFacadeService;
    }
}
```

‍
