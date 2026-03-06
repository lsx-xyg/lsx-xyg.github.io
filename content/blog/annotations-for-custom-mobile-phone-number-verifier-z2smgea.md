---
title: 自定义手机号校验器注解
slug: annotations-for-custom-mobile-phone-number-verifier-z2smgea
url: /post/annotations-for-custom-mobile-phone-number-verifier-z2smgea.html
date: '2026-02-24 16:36:32+08:00'
lastmod: '2026-03-06 19:36:21+08:00'
toc: true
isCJKLanguage: true
---



# 自定义手机号校验器注解

- **创建自定义注解**：定义一个注解，并指定目标和保留策略。
- **实现校验器**：创建一个实现`ConstraintValidator`接口的类。
- **将注解应用于字段**：在需要校验的字段上使用自定义注解。

## **创建自定义注解**

```java
package cn.hollis.nft.turbo.base.validator;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 是否手机号校验注解
 */
@Constraint(validatedBy = MobileValidator.class)
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface IsMobile {
    String message() default "手机号格式不正确"; // 默认错误信息

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}


```

## 实现校验器

```java
package cn.hollis.nft.turbo.base.validator;

import cn.hutool.core.lang.Validator;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

/**
 * 手机号校验器
 */
public class MobileValidator implements ConstraintValidator<IsMobile, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return Validator.isMobile(value);
    }
}
```

## 使用方法

```java
@GetMapping("/sendCaptcha")
public Result<Boolean> sendCaptcha(@IsMobile String telephone) {
    NoticeResponse noticeResponse = noticeFacadeService.generateAndSendSmsCaptcha(telephone);
    return Result.success(noticeResponse.getSuccess());
}
```

或者

```java
public class RegisterParam {
    /**
     * 手机号
     */
    @IsMobile
    private String telephone;
}
```
