---
title: 通用分布式注解实现
slug: universal-distributed-annotation-implementation-2le1wc
url: /post/universal-distributed-annotation-implementation-2le1wc.html
date: '2026-02-24 15:44:53+08:00'
lastmod: '2026-03-06 19:36:14+08:00'
toc: true
isCJKLanguage: true
---



# 通用分布式注解实现

## 分布式锁常量 —— DistributeLockConstant

```java
package *.lock;

/**
 * 分布式锁常量
 */
public class DistributeLockConstant {

    public static final String NONE_KEY = "NONE";

    public static final String DEFAULT_OWNER = "DEFAULT";

    public static final int DEFAULT_EXPIRE_TIME = -1;

    public static final int DEFAULT_WAIT_TIME = -1;
}
```

## 分布式锁注解 —— DistributeLock

```java
package *.lock;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 分布式锁注解
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributeLock {

    /**
     * 锁的场景
     *
     * @return
     */
    public String scene();

    /**
     * 加锁的key，优先取key()，如果没有，则取keyExpression()
     *
     * @return
     */
    public String key() default DistributeLockConstant.NONE_KEY;

    /**
     * SPEL表达式:
     * <pre>
     *     #id
     *     #insertResult.id
     * </pre>
     *
     * @return
     */
    public String keyExpression() default DistributeLockConstant.NONE_KEY;

    /**
     * 超时时间，毫秒
     * 默认情况下不设置超时时间，会自动续期
     *
     * @return
     */
    public int expireTime() default DistributeLockConstant.DEFAULT_EXPIRE_TIME;

    /**
     * 加锁等待时长，毫秒
     * 默认情况下不设置等待时长，不做等待
     * @return
     */
    public int waitTime() default DistributeLockConstant.DEFAULT_WAIT_TIME;
}
```

## 分布式锁切面 —— DistributeLockAspect

为什么使用 @Order(Integer.MIN_VALUE)[^1]

```java
package *.lock;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.StandardReflectionParameterNameDiscoverer;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.concurrent.TimeUnit;

/**
 * 分布式锁切面
 */
@Aspect
@Component
@Order(Integer.MIN_VALUE)
public class DistributeLockAspect {

    private RedissonClient redissonClient;

    public DistributeLockAspect(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    private static final Logger LOG = LoggerFactory.getLogger(DistributeLockAspect.class);

    @Around("@annotation(*.lock.DistributeLock)")
    public Object process(ProceedingJoinPoint pjp) throws Exception {
        Object response = null;
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        DistributeLock distributeLock = method.getAnnotation(DistributeLock.class);

        String key = distributeLock.key();
        if (DistributeLockConstant.NONE_KEY.equals(key)) {
			// key 和 keyExpression 都没配置
            if (DistributeLockConstant.NONE_KEY.equals(distributeLock.keyExpression())) {
                throw new DistributeLockException("no lock key found...");
            }
            SpelExpressionParser parser = new SpelExpressionParser();
            Expression expression = parser.parseExpression(distributeLock.keyExpression());

            EvaluationContext context = new StandardEvaluationContext();
            // 获取参数值
            Object[] args = pjp.getArgs();

            // 获取运行时参数的名称
            StandardReflectionParameterNameDiscoverer discoverer
                    = new StandardReflectionParameterNameDiscoverer();
            String[] parameterNames = discoverer.getParameterNames(method);

            // 将参数绑定到context中
            if (parameterNames != null) {
                for (int i = 0; i < parameterNames.length; i++) {
                    context.setVariable(parameterNames[i], args[i]);
                }
            }

            // 解析表达式，获取结果
            key = String.valueOf(expression.getValue(context));
        }

        String scene = distributeLock.scene();

        String lockKey = scene + "#" + key;

        int expireTime = distributeLock.expireTime();
        int waitTime = distributeLock.waitTime();
        RLock rLock = redissonClient.getLock(lockKey);
        boolean lockResult = false;
        if (waitTime == DistributeLockConstant.DEFAULT_WAIT_TIME) {
            if (expireTime == DistributeLockConstant.DEFAULT_EXPIRE_TIME) {
                LOG.info(String.format("lock for key : %s", lockKey));
                rLock.lock();
            } else {
                LOG.info(String.format("lock for key : %s , expire : %s", lockKey, expireTime));
                rLock.lock(expireTime, TimeUnit.MILLISECONDS);
            }
            lockResult = true;
        } else {
            if (expireTime == DistributeLockConstant.DEFAULT_EXPIRE_TIME) {
                LOG.info(String.format("try lock for key : %s , wait : %s", lockKey, waitTime));
                lockResult = rLock.tryLock(waitTime, TimeUnit.MILLISECONDS);
            } else {
                LOG.info(String.format("try lock for key : %s , expire : %s , wait : %s", lockKey, expireTime, waitTime));
                lockResult = rLock.tryLock(waitTime, expireTime, TimeUnit.MILLISECONDS);
            }
        }

        if (!lockResult) {
            LOG.warn(String.format("lock failed for key : %s , expire : %s", lockKey, expireTime));
            throw new DistributeLockException("acquire lock failed... key : " + lockKey);
        }

        try {
            LOG.info(String.format("lock success for key : %s , expire : %s", lockKey, expireTime));
            response = pjp.proceed();
        } catch (Throwable e) {
            throw new Exception(e);
        } finally {
            rLock.unlock();
            LOG.info(String.format("unlock for key : %s , expire : %s", lockKey, expireTime));
        }
        return response;
    }
}
```

## 分布式锁异常 —— DistributeLockException

```java
package cn.hollis.nft.turbo.lock;

/**
 * 分布式锁异常
 */
public class DistributeLockException extends RuntimeException {

    public DistributeLockException() {
    }

    public DistributeLockException(String message) {
        super(message);
    }

    public DistributeLockException(String message, Throwable cause) {
        super(message, cause);
    }

    public DistributeLockException(Throwable cause) {
        super(cause);
    }

    public DistributeLockException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
}
```

[^1]: # 问题汇总

    ## @DistributeLock 和 @Transactional 先后顺序

    切面上加了个 @Order(Integer.MIN\_VALUE)，让它可以最早执行，避免并发时事务未提交但是锁释放导致的问题

    ## 默认

    ```java
    1. 事务切面 (@Transactional)   ← 先执行
    2. 锁切面 (@DistributeLock)     ← 后执行

    执行顺序：
      事务开始
      锁获取
      业务逻辑
      锁释放    ← 问题在这里！
      事务提交  ← 锁已经释放了，但事务还没提交
    ```
    ### 添加 @Order

    [分布式锁切面 —— DistributeLockAspect](#20260224155008-e9hf5d8)

    ```java
    1. 锁切面 (@DistributeLock)     ← 最先执行（Integer.MIN_VALUE = -2147483648）
    2. 事务切面 (@Transactional)     ← 最后执行（默认 Integer.MAX_VALUE = 2147483647）

    执行顺序：
      锁获取
      事务开始
      业务逻辑
      事务提交  ← 先提交事务
      锁释放    ← 再释放锁
    ```
    ‍
