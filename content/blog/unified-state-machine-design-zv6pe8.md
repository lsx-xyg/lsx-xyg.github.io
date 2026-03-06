---
title: 统一状态机设计
slug: unified-state-machine-design-zv6pe8
url: /post/unified-state-machine-design-zv6pe8.html
date: '2026-02-24 14:22:58+08:00'
lastmod: '2026-03-06 19:35:40+08:00'
toc: true
isCJKLanguage: true
---



# 统一状态机设计

状态机：

1. 状态（States）：代表系统可能处于的各种状态，例如 "已下单"、"已支付"、"已发货" 等。
2. 事件（Events）：触发状态转换的事件，例如 "下单"、"支付"、"发货" 等。
3. 转换（Transitions）：定义状态之间的转换规则，即在某个事件发生时，系统从一个状态转换到另一个状态的规则。
4. 动作（Actions）：在状态转换发生时执行的操作或行为。

## StateMachine

```java
package *.base.statemachine;

public interface StateMachine<STATE, EVENT> {

    /**
     * 状态机转移
     *
     * @param state
     * @param event
     * @return
     */
    public STATE transition(STATE state, EVENT event);
}
```

## 状态机基类 —— BaseStateMachine

统一异常体系设计 —— ErrorCode[^1]

```java
package *.base.statemachine;

import *.base.exception.BizException;
import com.google.common.base.Joiner;
import com.google.common.collect.Maps;

import java.util.Map;

import static *.base.exception.BizErrorCode.STATE_MACHINE_TRANSITION_FAILED;

public class BaseStateMachine<STATE, EVENT> implements StateMachine<STATE, EVENT> {
    private Map<String, STATE> stateTransitions = Maps.newHashMap();

	// 原始状态_触发事件 目标状态
    protected void putTransition(STATE origin, EVENT event, STATE target) {
        stateTransitions.put(Joiner.on("_").join(origin, event), target);
    }

	// 检查 stateTransitions 中是否有状态&事件对
    @Override
    public STATE transition(STATE state, EVENT event) {
        STATE target = stateTransitions.get(Joiner.on("_").join(state, event));
        if (target == null) {
            throw new BizException("state = " + state + " , event = " + event, STATE_MACHINE_TRANSITION_FAILED);
        }
        return target;
    }
}
```

## 订单状态机 —— OrderStateMachine

```java
package *.order.domain.entity.statemachine;

import *.api.order.constant.TradeOrderEvent;
import *.api.order.constant.TradeOrderState;
import *.base.statemachine.BaseStateMachine;

public class OrderStateMachine extends BaseStateMachine<TradeOrderState, TradeOrderEvent> {

    public static final OrderStateMachine INSTANCE = new OrderStateMachine();

    {
        putTransition(TradeOrderState.CREATE, TradeOrderEvent.CONFIRM, TradeOrderState.CONFIRM);
        putTransition(TradeOrderState.CONFIRM, TradeOrderEvent.PAY, TradeOrderState.PAID);
        //库存预扣减成功，但是未真正扣减成功，也能支付/取消，不能因为延迟导致用户无法支付/取消。
        putTransition(TradeOrderState.CREATE, TradeOrderEvent.PAY, TradeOrderState.PAID);
        putTransition(TradeOrderState.CREATE, TradeOrderEvent.CANCEL, TradeOrderState.CLOSED);
        putTransition(TradeOrderState.CREATE, TradeOrderEvent.TIME_OUT, TradeOrderState.CLOSED);

        //已支付后，再确认，状态不变
        putTransition(TradeOrderState.PAID, TradeOrderEvent.CONFIRM, TradeOrderState.PAID);

        putTransition(TradeOrderState.CONFIRM, TradeOrderEvent.CANCEL, TradeOrderState.CLOSED);
        putTransition(TradeOrderState.CONFIRM, TradeOrderEvent.TIME_OUT, TradeOrderState.CLOSED);

        putTransition(TradeOrderState.PAID, TradeOrderEvent.FINISH, TradeOrderState.FINISH);
    }

}
```

‍

### 订单事件 —— TradeOrderEvent

```java
package *.api.order.constant;

/**
 * 订单事件
 */
public enum TradeOrderEvent {

    /**
     * 订单创建
     */
    CREATE,

    /**
     * 订单确认
     */
    CONFIRM,

    /**
     * 订单支付
     */
    PAY,

    /**
     * 订单取消
     */
    CANCEL,

    /**
     * 订单超时
     */
    TIME_OUT,

    /**
     * 订单完成
     */
    FINISH;
}

```

### 订单状态 —— TradeOrderState

```java
package *.api.order.constant;

/**
 * 订单状态
 */
public enum TradeOrderState {

    /**
     * 订单创建
     */
    CREATE,

    /**
     * 订单确认
     */
    CONFIRM,

    /**
     * 已付款
     */
    PAID,

    /**
     * 交易成功
     */
    FINISH,

    /**
     * 订单关闭
     */
    CLOSED;
}
```

### 订单错误码 —— OrderErrorCode

```java
package *.api.order.constant;

import *.base.exception.ErrorCode;

public enum OrderErrorCode implements ErrorCode {
    /**
     * 订单不存在
     */
    ORDER_NOT_EXIST("ORDER_NOT_EXIST", "订单不存在"),

    /**
     * 无权限操作
     */
    PERMISSION_DENIED("PERMISSION_DENIED", "无权限操作"),

    /**
     * 更新订单失败
     */
    UPDATE_ORDER_FAILED("UPDATE_ORDER_FAILED", "更新订单失败"),

    /**
     * 订单已支付
     */
    ORDER_ALREADY_PAID("ORDER_ALREADY_PAID", "订单已支付"),

    /**
     * 订单状态转移非法
     */
    ORDER_STATE_TRANSFER_ILLEGAL("ORDER_STATE_TRANSFER_ILLEGAL", "订单状态转移非法"),

    /**
     * 库存扣件失败
     */
    INVENTORY_DEDUCT_FAILED("INVENTORY_DEDUCT_FAILED", "库存扣减失败"),

    /**
     * 订单创建校验失败
     */
    ORDER_CREATE_VALID_FAILED("ORDER_CREATE_VALID_FAILED", "订单创建校验失败"),
    
	/**
     * 订单已过期
     */
    ORDER_IS_EXPIRED("OEDER_IS_EXPIRED", "订单已过期"),

    /**
     * 买家不能是平台用户
     */
    BUYER_IS_PLATFORM_USER("BUYER_IS_PLATFORM_USER", "买家不能是平台用户"),

    /**
     * 买家状态异常
     */
    BUYER_STATUS_ABNORMAL("BUYER_STATUS_ABNORMAL", "买家状态异常"),
    
	/**
     * 买家未完成实名认证
     */
    BUYER_NOT_AUTH("BUYER_NOT_AUTH", "买家未完成实名认证"),

    /**
     * 库存不足
     */
    INVENTORY_NOT_ENOUGH("INVENTORY_NOT_ENOUGH", "库存不足"),

    /**
     * 商品不可用
     */
    GOODS_NOT_AVAILABLE("GOODS_NOT_AVAILABLE", "商品不可用"),

    /**
     * 商品价格发生变化
     */
    GOODS_PRICE_CHANGED("GOODS_PRICE_CHANGED", "商品价格发生变化");


    private String code;

    private String message;

    OrderErrorCode(String code, String message) {
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

[^1]: # 统一异常体系设计

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
