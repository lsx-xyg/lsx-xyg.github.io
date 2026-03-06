---
title: 统一入参合法性校验(RPC)
slug: unified-input-parameter-validity-verification-rpc-z2g124v
url: /post/unified-input-parameter-validity-verification-rpc-z2g124v.html
date: '2026-02-24 14:58:17+08:00'
lastmod: '2026-03-06 19:32:50+08:00'
toc: true
isCJKLanguage: true
---



# 统一入参合法性校验(RPC)

RPC 调用参数校验自定义注解使用 AOP

## 自定义注解 —— @Facade

```java
package *.rpc.facade;

public @interface Facade {
}
```

## AOP 处理逻辑

FacadeAspect 完整代码[^1]

```java
package *.rpc.facade;

/**
 * Facade的切面处理类，统一统计进行参数校验及异常捕获
 */
@Aspect
@Component
public class FacadeAspect {

    private static final Logger LOGGER = LoggerFactory.getLogger(FacadeAspect.class);

    @Around("@annotation(*.rpc.facade.Facade)")
    public Object facade(ProceedingJoinPoint pjp) throws Exception {

        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        Object[] args = pjp.getArgs();
        LOGGER.info("start to execute , method = " + method.getName() + " , args = " + JSON.toJSONString(args));

        Class returnType = ((MethodSignature) pjp.getSignature()).getMethod().getReturnType();

        //循环遍历所有参数，进行参数校验
        for (Object parameter : args) {
            try {
                BeanValidator.validateObject(parameter);
            } catch (ValidationException e) {
                printLog(stopWatch, method, args, "failed to validate", null, e);
                return getFailedResponse(returnType, e);
            }
        }

        //省略其他代码
    }
   
}
```

## BeanValidator

```java
package *.base.utils;


/**
 * 参数校验工具
 */
public class BeanValidator {

	// 一定要定义成 static 的变量，而不是在方法内每次都创建一个 ，会导致 CPU 飙高
    private static Validator validator = Validation.byProvider(HibernateValidator.class)
										.configure().failFast(true)
            							.buildValidatorFactory().getValidator();

    /**
     * @param object object
     * @param groups groups
     */
    public static void validateObject(Object object, Class<?>... groups) throws ValidationException {
		// 验证失败的信息
        Set<ConstraintViolation<Object>> constraintViolations = validator.validate(object, groups);
        if (constraintViolations.stream().findFirst().isPresent()) {
            throw new ValidationException(constraintViolations.stream().findFirst().get().getMessage());
        }
    }
}
```

## 使用方法

```java
@DubboService(version = "1.0.0")
public class OrderFacadeServiceImpl implements OrderFacadeService {

    @Override
    @Facade // 在此处添加注解即可
    public OrderResponse create(OrderCreateRequest request) {
        Boolean preDeductResult = inventoryWrapperService.preDeduct(request);
        if (preDeductResult) {
            return orderService.create(request);
        }
        throw new OrderException(OrderErrorCode.INVENTORY_DEDUCT_FAILED);
    }
}
```

[^1]: # FacadeAspect 完整代码

    ```java
    package *.rpc.facade;

    import *.base.exception.BizException;
    import *.base.exception.SystemException;
    import *.base.response.BaseResponse;
    import *.base.response.ResponseCode;
    import *.base.utils.BeanValidator;
    import com.alibaba.fastjson2.JSON;
    import jakarta.validation.ValidationException;
    import org.apache.commons.lang3.StringUtils;
    import org.apache.commons.lang3.time.StopWatch;
    import org.aspectj.lang.ProceedingJoinPoint;
    import org.aspectj.lang.annotation.Around;
    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.reflect.MethodSignature;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.stereotype.Component;

    import java.lang.reflect.InvocationTargetException;
    import java.lang.reflect.Method;
    import java.util.Arrays;

    /**
     * Facade的切面处理类，统一统计进行参数校验及异常捕获
     */
    @Aspect
    @Component
    public class FacadeAspect {

        private static final Logger LOGGER = LoggerFactory.getLogger(FacadeAspect.class);

        @Around("@annotation(*.rpc.facade.Facade)")
        public Object facade(ProceedingJoinPoint pjp) throws Exception {

            StopWatch stopWatch = new StopWatch();
            stopWatch.start();

            Method method = ((MethodSignature) pjp.getSignature()).getMethod();
            Object[] args = pjp.getArgs();
            LOGGER.info("start to execute , method = " + method.getName() + " , args = " + JSON.toJSONString(args));

            Class returnType = ((MethodSignature) pjp.getSignature()).getMethod().getReturnType();

            // 循环遍历所有参数，进行参数校验
            for (Object parameter : args) {
                try {
                    BeanValidator.validateObject(parameter);
                } catch (ValidationException e) {
                    printLog(stopWatch, method, args, "failed to validate", null, e);
                    return getFailedResponse(returnType, e);
                }
            }

            try {
                // 目标方法执行
                Object response = pjp.proceed();
                enrichObject(response);
                printLog(stopWatch, method, args, "end to execute", response, null);
                return response;
            } catch (Throwable throwable) {
                // 如果执行异常，则返回一个失败的response
                printLog(stopWatch, method, args, "failed to execute", null, throwable);
                return getFailedResponse(returnType, throwable);
            }
        }

        /**
         * 日志打印
         *
         * @param stopWatch
         * @param method
         * @param args
         * @param action
         * @param response
         */
        private void printLog(StopWatch stopWatch, Method method, Object[] args, String action, Object response,
                              Throwable throwable) {
            try {
                //因为此处有JSON.toJSONString，可能会有异常，需要进行捕获，避免影响主干流程
                LOGGER.info(getInfoMessage(action, stopWatch, method, args, response, throwable), throwable);
                // 如果校验失败，则返回一个失败的response
            } catch (Exception e1) {
                LOGGER.error("log failed", e1);
            }
        }

        /**
         * 统一格式输出，方便做日志统计
         * <p>
         * *** 如果调整此处的格式，需要同步调整日志监控 ***
         *
         * @param action    行为
         * @param stopWatch 耗时
         * @param method    方法
         * @param args      参数
         * @param response  响应
         * @return 拼接后的字符串
         */
        private String getInfoMessage(String action, StopWatch stopWatch, Method method, Object[] args, Object response,
                                      Throwable exception) {

            StringBuilder stringBuilder = new StringBuilder(action);
            stringBuilder.append(" ,method = ");
            stringBuilder.append(method.getName());
            stringBuilder.append(" ,cost = ");
            stringBuilder.append(stopWatch.getTime()).append(" ms");
            if (response instanceof BaseResponse) {
                stringBuilder.append(" ,success = ");
                stringBuilder.append(((BaseResponse) response).getSuccess());
            }
            if (exception != null) {
                stringBuilder.append(" ,success = ");
                stringBuilder.append(false);
            }
            stringBuilder.append(" ,args = ");
            stringBuilder.append(JSON.toJSONString(Arrays.toString(args)));

            if (response != null) {
                stringBuilder.append(" ,resp = ");
                stringBuilder.append(JSON.toJSONString(response));
            }

            if (exception != null) {
                stringBuilder.append(" ,exception = ");
                stringBuilder.append(exception.getMessage());
            }

            if (response instanceof BaseResponse) {
                BaseResponse baseResponse = (BaseResponse) response;
                if (!baseResponse.getSuccess()) {
                    stringBuilder.append(" , execute_failed");
                }
            }

            return stringBuilder.toString();
        }

        /**
         * 将 response 的信息补全，主要是 code 和 message
         *
         * @param response
         */
        private void enrichObject(Object response) {
            if (response instanceof BaseResponse) {
                if (((BaseResponse) response).getSuccess()) {
                    // 如果状态是成功的，需要将未设置的 responseCode 设置成 SUCCESS
                    if (StringUtils.isEmpty(((BaseResponse) response).getResponseCode())) {
                        ((BaseResponse) response).setResponseCode(ResponseCode.SUCCESS.name());
                    }
                } else {
                    // 如果状态是成功的，需要将未设置的 responseCode 设置成 BIZ_ERROR
                    if (StringUtils.isEmpty(((BaseResponse) response).getResponseCode())) {
                        ((BaseResponse) response).setResponseCode(ResponseCode.BIZ_ERROR.name());
                    }
                }
            }
        }

        /**
         * 定义并返回一个通用的失败响应
         */
        private Object getFailedResponse(Class returnType, Throwable throwable)
                throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {

            // 如果返回值的类型为BaseResponse 的子类，则创建一个通用的失败响应
            if (returnType.getDeclaredConstructor().newInstance() instanceof BaseResponse) {
                BaseResponse response = (BaseResponse) returnType.getDeclaredConstructor().newInstance();
                response.setSuccess(false);
                if (throwable instanceof BizException bizException) {
                    response.setResponseMessage(bizException.getErrorCode().getMessage());
                    response.setResponseCode(bizException.getErrorCode().getCode());
                } else if (throwable instanceof SystemException systemException) {
                    response.setResponseMessage(systemException.getErrorCode().getMessage());
                    response.setResponseCode(systemException.getErrorCode().getCode());
                } else {
                    response.setResponseMessage(throwable.toString());
                    response.setResponseCode(ResponseCode.BIZ_ERROR.name());
                }

                return response;
            }

            LOGGER.error(
                    "failed to getFailedResponse , returnType (" + returnType + ") is not instanceof BaseResponse");
            return null;
        }
    }
    ```
