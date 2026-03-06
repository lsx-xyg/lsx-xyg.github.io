---
title: Controller 统一异常处理
slug: controller-unified-exception-handling-z1t1kdt
url: /post/controller-unified-exception-handling-z1t1kdt.html
date: '2026-02-24 15:10:39+08:00'
lastmod: '2026-03-06 19:35:51+08:00'
toc: true
isCJKLanguage: true
---



# Controller 统一异常处理

基于 @ControllerAdvice

```java
package *.web.handler;

import *.base.exception.BizException;
import *.base.exception.SystemException;
import *.web.vo.Result;
import com.google.common.collect.Maps;
import org.springframework.http.HttpStatus;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;

import java.util.Map;

@ControllerAdvice
public class GlobalWebExceptionHandler {

    /**
     * 自定义方法参数校验异常处理器
     *
     * @param ex
     * @return
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ResponseBody
    public Map<String, String> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = Maps.newHashMapWithExpectedSize(1);
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return errors;
    }

    /**
     * 自定义业务异常处理器
     *
     * @param bizException
     * @return
     */
    @ExceptionHandler(BizException.class)
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public Result exceptionHandler(BizException bizException) {
        Result result = new Result();
        result.setCode(bizException.getErrorCode().getCode());
        result.setMessage(bizException.getErrorCode().getMessage());
        result.setSuccess(false);
        return result;
    }

    /**
     * 自定义系统异常处理器
     *
     * @param systemException
     * @return
     */
    @ExceptionHandler(SystemException.class)
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public Result systemExceptionHandler(SystemException systemException) {
        Result result = new Result();
        result.setCode(systemException.getErrorCode().getCode());
        result.setMessage(systemException.getErrorCode().getMessage());
        result.setSuccess(false);
        return result;
    }
}
```

‍
