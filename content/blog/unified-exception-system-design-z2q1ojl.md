---
title: 统一异常体系设计
slug: unified-exception-system-design-z2q1ojl
url: /post/unified-exception-system-design-z2q1ojl.html
date: '2026-02-24 14:16:43+08:00'
lastmod: '2026-03-06 19:32:38+08:00'
toc: true
isCJKLanguage: true
---



# 统一异常体系设计

1. 系统异常
2. 业务异常

## ErrorCode 及具体实现

```java
package *.base.exception;

/**
 * 错误码
 */
public interface ErrorCode {
    /**
     * 错误码
     */
    String getCode();

    /**
     * 错误信息
     */
    String getMessage();
}
```

```java
package *.trade.exception;

import *.base.exception.ErrorCode;

public enum TradeErrorCode implements ErrorCode {

    /**
     * 订单创建失败
     */
    ORDER_CREATE_FAILED("ORDER_CREATE_FAILED", "订单创建失败"),

    /**
     * 无支付权限
     */
    PAY_PERMISSION_DENIED("PAY_PERMISSION_DENIED", "无支付权限"),

    /**
     * 支付创建失败
     */
    PAY_CREATE_FAILED("PAY_CREATE_FAILED", "支付创建失败"),

    /**
     * 商品不可售卖
     */
    GOODS_NOT_FOR_SALE("GOODS_NOT_FOR_SALE", "商品不可售卖"),

    /**
     * 商品不存在
     */
    GOODS_NOT_EXIST("GOODS_NOT_EXIST", "商品不存在"),

    /**
     * 订单不可支付
     */
    ORDER_IS_CANNOT_PAY("ORDER_IS_CANNOT_PAY", "订单不可支付"),

    /**
     * 订单取消失败
     */
    ORDER_CANCEL_FAILED("ORDER_CANCEL_FAILED", "订单取消失败");

    private String code;

    private String message;

    TradeErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }

    @Override
    public String getCode() {
        return this.code;
    }

    @Override
    public String getMessage() {
        return this.message;
    }
}
```

## 业务异常 —— BizException

```java
package *.base.exception;

/**
 * 业务异常
 */
public class BizException extends RuntimeException {

    private ErrorCode errorCode;

    public BizException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public BizException(String message, ErrorCode errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public BizException(String message, Throwable cause, ErrorCode errorCode) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public BizException(Throwable cause, ErrorCode errorCode) {
        super(cause);
        this.errorCode = errorCode;
    }

    public BizException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace, ErrorCode errorCode) {
        super(message, cause, enableSuppression, writableStackTrace);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }

    public void setErrorCode(ErrorCode errorCode) {
        this.errorCode = errorCode;
    }
}
```

‍
