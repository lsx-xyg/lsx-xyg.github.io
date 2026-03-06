---
title: 通用入参出参设计
slug: universal-input-and-output-parameters-design-z1tfibi
url: /post/universal-input-and-output-parameters-design-z1tfibi.html
date: '2026-02-24 13:59:20+08:00'
lastmod: '2026-03-06 19:39:38+08:00'
toc: true
isCJKLanguage: true
---





## 入参基类 —— BaseRequest

```java
package *.base.request;

import lombok.Getter;
import lombok.Setter;

import java.io.Serializable;

@Setter
@Getter
public class BaseRequest implements Serializable {
    private static final long serialVersionUID = 1L;   
}
```

## 出参基类 —— BaseResponse

```java
package *.base.response;

import lombok.Getter;
import lombok.Setter;

import java.io.Serializable;
@Setter
@Getter
public class BaseResponse implements Serializable {
    private static final long serialVersionUID = 1L;

    private Boolean success;

    private String responseCode;

    private String responseMessage;
}
```

## 分页类 —— PageRequest、PageResponse

```java
package *.base.request;

import lombok.Getter;
import lombok.Setter;

@Setter
@Getter
public class PageRequest extends BaseRequest {
    private static final long serialVersionUID = 1L;

    /**
     * 当前页
     */
    private int currentPage;

    /**
     * 每页结果数
     */
    private int pageSize;
}
```

```java
package *.base.response;

import lombok.Getter;
import lombok.Setter;

import java.util.List;

@Setter
@Getter
public class PageResponse<T> extends MultiResponse<T> {
    private static final long serialVersionUID = 1L;

    /**
     * 当前页
     */
    private int currentPage;

    /**
     * 每页结果数
     */
    private int pageSize;

    /**
     * 总页数
     */
    private int totalPage;

    /**
     * 总数
     */
    private int total;

    public static <T> PageResponse<T> of(List<T> datas, int total, int pageSize) {
        PageResponse<T> multiResponse = new PageResponse<>();
        multiResponse.setSuccess(true);
        multiResponse.setDatas(datas);
        multiResponse.setTotal(total);
        multiResponse.setPageSize(pageSize);
        multiResponse.setTotalPage((pageSize + total - 1) / pageSize);
        return multiResponse;
    }
}
```

## 单条数据类 —— SingleResponse

```java
package *.base.response;

import lombok.Getter;
import lombok.Setter;

@Setter
@Getter
public class SingleResponse<T> extends BaseResponse {
    private static final long serialVersionUID = 1L;

    private T data;

    public static <T> SingleResponse<T> of(T data) {
        SingleResponse<T> singleResponse = new SingleResponse<>();
        singleResponse.setSuccess(true);
        singleResponse.setData(data);
        return singleResponse;
    }

    public static <T> SingleResponse<T> fail(String errorCode, String errorMessage) {
        SingleResponse<T> singleResponse = new SingleResponse<>();
        singleResponse.setSuccess(false);
        singleResponse.setResponseCode(errorCode);
        singleResponse.setResponseMessage(errorMessage);
        return singleResponse;
    }

}

```

## 多条数据类 —— MultiResponse

```java
package *.base.response;

import lombok.Getter;
import lombok.Setter;

import java.io.Serializable;
import java.util.List;

@Setter
@Getter
public class MultiResponse<T> extends BaseResponse implements Serializable {
    private static final long serialVersionUID = 1L;

    private List<T> datas;

    public static <T> MultiResponse<T> of(List<T> datas) {
        MultiResponse<T> multiResponse = new MultiResponse<>();
        multiResponse.setSuccess(true);
        multiResponse.setDatas(datas);
        return multiResponse;
    }

}
```

## 响应码 —— ResponseCode

```java
package *.base.response;

public enum ResponseCode {

    /**
     * 成功
     */
    SUCCESS,

    /**
     * 重复
     */
    DUPLICATED,

    /**
     * 非法参数
     */
    ILLEGAL_ARGUMENT,

    /**
     * 系统错误
     */
    SYSTEM_ERROR,

    /**
     * 业务错误
     */
    BIZ_ERROR;
}
```

## Rest响应 —— RestResponse

```java
package *.base.response;

import com.alibaba.fastjson2.JSONObject;

import lombok.Getter;
import lombok.Setter;

@Setter
@Getter
public class RestResponse extends BaseResponse {

    private JSONObject data;

    private JSONObject error;

    @Override
    public Boolean getSuccess() {
        return data != null;
    }

    @Override
    public String getResponseMessage() {
        if (this.error != null) {
            return error.getString("message");
        }
        return null;
    }

    @Override
    public String getResponseCode() {
        if (this.error != null) {
            return error.getString("code");
        }
        return null;
    }
}

```
