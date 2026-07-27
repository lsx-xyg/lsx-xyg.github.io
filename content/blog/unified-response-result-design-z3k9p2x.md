---
title: 统一接口响应结果设计
slug: unified-response-result-design-z3k9p2x
url: /post/unified-response-result-design-z3k9p2x.html
date: '2026-07-27 10:00:00+08:00'
lastmod: '2026-07-27 10:00:00+08:00'
toc: true
isCJKLanguage: true
---

# 统一接口响应结果设计

在前后端分离的架构中，统一的接口响应格式是提升协作效率、降低沟通成本的关键。本文将从实际项目出发，介绍一套完整的统一响应结果设计方案。

## 为什么需要统一响应

1. **降低沟通成本**：前后端按照统一格式对接，无需每次协商返回结构
2. **统一异常处理**：业务异常和系统异常使用相同的返回格式
3. **便于全局处理**：可以通过拦截器或 AOP 统一包装响应结果
4. **提升代码复用**：通用的响应工具类可以在各模块中复用

## Result 统一响应类设计

### 基础结构

```java
package *.base.response;

import java.io.Serializable;

/**
 * 统一响应结果
 */
public class Result<T> implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 响应码
     */
    private String code;

    /**
     * 响应消息
     */
    private String message;

    /**
     * 响应数据
     */
    private T data;

    /**
     * 响应时间戳
     */
    private Long timestamp;

    public Result() {
        this.timestamp = System.currentTimeMillis();
    }

    public Result(String code, String message, T data) {
        this.code = code;
        this.message = message;
        this.data = data;
        this.timestamp = System.currentTimeMillis();
    }

    /**
     * 成功响应（无数据）
     */
    public static <T> Result<T> success() {
        return new Result<>(ResultCode.SUCCESS.getCode(), ResultCode.SUCCESS.getMessage(), null);
    }

    /**
     * 成功响应（带数据）
     */
    public static <T> Result<T> success(T data) {
        return new Result<>(ResultCode.SUCCESS.getCode(), ResultCode.SUCCESS.getMessage(), data);
    }

    /**
     * 成功响应（带消息和数据）
     */
    public static <T> Result<T> success(String message, T data) {
        return new Result<>(ResultCode.SUCCESS.getCode(), message, data);
    }

    /**
     * 失败响应
     */
    public static <T> Result<T> fail(String code, String message) {
        return new Result<>(code, message, null);
    }

    /**
     * 失败响应（使用错误码枚举）
     */
    public static <T> Result<T> fail(ResultCode resultCode) {
        return new Result<>(resultCode.getCode(), resultCode.getMessage(), null);
    }

    /**
     * 失败响应（带错误码和自定义消息）
     */
    public static <T> Result<T> fail(ResultCode resultCode, String message) {
        return new Result<>(resultCode.getCode(), message, null);
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }

    public Long getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Long timestamp) {
        this.timestamp = timestamp;
    }
}
```

### 响应码枚举

```java
package *.base.response;

/**
 * 响应码枚举
 */
public enum ResultCode {

    /**
     * 成功
     */
    SUCCESS("0", "操作成功"),

    /**
     * 系统异常
     */
    SYSTEM_ERROR("SYSTEM_ERROR", "系统异常，请稍后重试"),

    /**
     * 参数校验失败
     */
    PARAM_VALIDATE_FAILED("PARAM_VALIDATE_FAILED", "参数校验失败"),

    /**
     * 请求参数格式错误
     */
    PARAM_FORMAT_ERROR("PARAM_FORMAT_ERROR", "请求参数格式错误"),

    /**
     * 未授权
     */
    UNAUTHORIZED("UNAUTHORIZED", "未授权，请先登录"),

    /**
     * 无权限访问
     */
    PERMISSION_DENIED("PERMISSION_DENIED", "无权限访问"),

    /**
     * 请求不存在
     */
    NOT_FOUND("NOT_FOUND", "请求的资源不存在"),

    /**
     * 请求方式不支持
     */
    METHOD_NOT_ALLOWED("METHOD_NOT_ALLOWED", "请求方式不支持"),

    /**
     * 操作频繁
     */
    TOO_MANY_REQUESTS("TOO_MANY_REQUESTS", "操作过于频繁，请稍后重试"),

    /**
     * 业务异常
     */
    BUSINESS_ERROR("BUSINESS_ERROR", "业务处理失败");

    private final String code;
    private final String message;

    ResultCode(String code, String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }
}
```

## 分页响应封装

对于列表查询接口，通常需要返回分页信息，我们可以单独封装分页响应。

### PageResult 分页结果

```java
package *.base.response;

import java.io.Serializable;
import java.util.List;

/**
 * 分页响应结果
 */
public class PageResult<T> implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 当前页码
     */
    private Integer pageNum;

    /**
     * 每页大小
     */
    private Integer pageSize;

    /**
     * 总记录数
     */
    private Long total;

    /**
     * 总页数
     */
    private Integer pages;

    /**
     * 数据列表
     */
    private List<T> list;

    public PageResult() {
    }

    public PageResult(Integer pageNum, Integer pageSize, Long total, List<T> list) {
        this.pageNum = pageNum;
        this.pageSize = pageSize;
        this.total = total;
        this.list = list;
        this.pages = (int) Math.ceil((double) total / pageSize);
    }

    /**
     * 构建分页结果
     */
    public static <T> PageResult<T> of(Integer pageNum, Integer pageSize, Long total, List<T> list) {
        return new PageResult<>(pageNum, pageSize, total, list);
    }

    /**
     * 空结果
     */
    public static <T> PageResult<T> empty(Integer pageNum, Integer pageSize) {
        return new PageResult<>(pageNum, pageSize, 0L, null);
    }

    public Integer getPageNum() {
        return pageNum;
    }

    public void setPageNum(Integer pageNum) {
        this.pageNum = pageNum;
    }

    public Integer getPageSize() {
        return pageSize;
    }

    public void setPageSize(Integer pageSize) {
        this.pageSize = pageSize;
    }

    public Long getTotal() {
        return total;
    }

    public void setTotal(Long total) {
        this.total = total;
    }

    public Integer getPages() {
        return pages;
    }

    public void setPages(Integer pages) {
        this.pages = pages;
    }

    public List<T> getList() {
        return list;
    }

    public void setList(List<T> list) {
        this.list = list;
    }
}
```

## 全局响应包装

通过 ResponseBodyAdvice 可以实现对 Controller 返回结果的统一包装，无需每个方法手动封装。

### 全局响应处理器

```java
package *.base.handler;

import *.base.response.Result;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

/**
 * 全局响应包装处理器
 */
@ControllerAdvice(basePackages = "*.controller")
public class GlobalResponseHandler implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // 如果返回类型已经是 Result，则不包装
        return !Result.class.isAssignableFrom(returnType.getParameterType());
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {
        // 空返回直接返回成功
        if (body == null) {
            return Result.success();
        }
        // 如果已经是 Result 类型，直接返回
        if (body instanceof Result) {
            return body;
        }
        // String 类型需要特殊处理，避免类型转换异常
        if (body instanceof String) {
            return body;
        }
        // 其他类型统一包装为 Result.success(data)
        return Result.success(body);
    }
}
```

### 忽略统一包装注解

对于某些特殊接口（如对接第三方回调），可能需要跳过统一包装，我们可以通过自定义注解来控制。

```java
package *.base.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 忽略统一响应包装
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface IgnoreResponseWrap {
}
```

在 GlobalResponseHandler 中增加判断：

```java
@Override
public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
    // 检查方法或类上是否有 IgnoreResponseWrap 注解
    if (returnType.hasMethodAnnotation(IgnoreResponseWrap.class)
            || returnType.getDeclaringClass().isAnnotationPresent(IgnoreResponseWrap.class)) {
        return false;
    }
    return !Result.class.isAssignableFrom(returnType.getParameterType());
}
```

## 统一异常与响应结合

将统一异常体系与响应结果结合，可以实现异常的统一捕获和格式化响应。

### 全局异常处理器

```java
package *.base.handler;

import *.base.exception.BizException;
import *.base.response.Result;
import *.base.response.ResultCode;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.validation.BindException;
import org.springframework.validation.FieldError;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.NoHandlerFoundException;

import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;
import java.util.stream.Collectors;

/**
 * 全局异常处理器
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /**
     * 处理业务异常
     */
    @ExceptionHandler(BizException.class)
    public Result<Void> handleBizException(BizException e) {
        log.warn("业务异常：{}", e.getMessage());
        return Result.fail(e.getErrorCode().getCode(), e.getMessage());
    }

    /**
     * 处理参数校验异常（@Valid @RequestBody）
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));
        log.warn("参数校验失败：{}", message);
        return Result.fail(ResultCode.PARAM_VALIDATE_FAILED, message);
    }

    /**
     * 处理参数绑定异常（@Valid @ModelAttribute）
     */
    @ExceptionHandler(BindException.class)
    public Result<Void> handleBindException(BindException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));
        log.warn("参数绑定失败：{}", message);
        return Result.fail(ResultCode.PARAM_VALIDATE_FAILED, message);
    }

    /**
     * 处理约束违反异常（@Validated 路径参数）
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public Result<Void> handleConstraintViolationException(ConstraintViolationException e) {
        String message = e.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining("; "));
        log.warn("约束违反：{}", message);
        return Result.fail(ResultCode.PARAM_VALIDATE_FAILED, message);
    }

    /**
     * 处理请求参数格式错误
     */
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public Result<Void> handleHttpMessageNotReadableException(HttpMessageNotReadableException e) {
        log.warn("请求参数格式错误：{}", e.getMessage());
        return Result.fail(ResultCode.PARAM_FORMAT_ERROR);
    }

    /**
     * 处理404异常
     */
    @ExceptionHandler(NoHandlerFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Result<Void> handleNoHandlerFoundException(NoHandlerFoundException e) {
        log.warn("请求路径不存在：{}", e.getRequestURL());
        return Result.fail(ResultCode.NOT_FOUND);
    }

    /**
     * 处理请求方式不支持
     */
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public Result<Void> handleHttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) {
        log.warn("请求方式不支持：{}", e.getMethod());
        return Result.fail(ResultCode.METHOD_NOT_ALLOWED);
    }

    /**
     * 处理其他未知异常
     */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(ResultCode.SYSTEM_ERROR);
    }
}
```

## 使用示例

### Controller 层使用

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    /**
     * 查询用户详情（自动包装）
     */
    @GetMapping("/{id}")
    public UserVO getUserById(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    /**
     * 分页查询用户列表（手动包装分页结果）
     */
    @GetMapping("/list")
    public Result<PageResult<UserVO>> listUsers(@RequestParam(defaultValue = "1") Integer pageNum,
                                                @RequestParam(defaultValue = "10") Integer pageSize,
                                                UserQuery query) {
        PageResult<UserVO> pageResult = userService.listUsers(pageNum, pageSize, query);
        return Result.success(pageResult);
    }

    /**
     * 新增用户
     */
    @PostMapping
    public void addUser(@RequestBody @Valid UserAddDTO dto) {
        userService.addUser(dto);
    }

    /**
     * 第三方回调接口（跳过统一包装）
     */
    @IgnoreResponseWrap
    @PostMapping("/callback")
    public String callback(@RequestBody CallbackDTO dto) {
        userService.handleCallback(dto);
        return "success";
    }
}
```

### Service 层抛出业务异常

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserVO getUserById(Long id) {
        User user = userMapper.selectById(id);
        if (user == null) {
            throw new BizException(UserErrorCode.USER_NOT_EXIST);
        }
        return convertToVO(user);
    }
}
```

## 最佳实践

1. **成功响应统一使用 Result.success()**，尽量通过全局包装自动完成
2. **业务异常统一抛出 BizException**，由全局异常处理器转换为响应结果
3. **错误码按模块管理**，各业务域定义自己的错误码枚举，实现统一的 ErrorCode 接口
4. **分页接口统一返回 PageResult**，并包装在 Result 中
5. **敏感信息脱敏**，响应数据中的手机号、身份证等敏感信息需要脱敏处理
6. **响应日志记录**，通过 AOP 或拦截器统一记录请求响应日志，便于排查问题

通过以上设计，可以让整个项目的接口风格保持一致，提升代码的可维护性和团队协作效率。
