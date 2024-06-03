---
title: Java中的校验：Jsr 303规范和Spring的适配扩展
categories: [编程, Java ]
---


请区分规范和实现

名词解释：

- *Jsr 303*：*Jsr 303*规范，规范代码在`javax.validation`包
- *Spring Validation*：Spring Framework中数据绑定和校验实现，目的是适配*Jsr 303*到Spring体系中，并对*Jsr 303*
  做了扩展，代码在`org.springframework.validation`包，Designed for convenient use with Spring's JSR-303 support but not
  JSR-303 specific。
- *Hibernate Validator*：Jsr303规范的实现，pom依赖`hibernate-validator`
- *Spring Web*：spring web借助*Spring Validation*完成对请求体和响应体的校验。需要兼容*Jsr303*规范，但*Spring Web*没有提供对
  *Jsr303*规范的实现
- *Spring Boot* ：spring-boot-starter-validation 和 spring-boot-starter-web ，web包提供了*Jsr303*
  的实现`hibernate-validator`

# JSR 303

原文请参考 [JSR 303: Bean Validation](https://beanvalidation.org/1.0/spec/)。


> Goals: To **avoid duplication** of these validations from the presentation layer to the persistence layer. This JSR
> defines a metadata model and API for JavaBean validation. **not limited to web tier or persistence tier**, available for
> both server-side application and client Swing application.

该规范主要包含以下几个方面：

- Constraint Definition
- Constraint declaration and validation process
- Validation APIs : how to programmatically validate a JavaBean
- Constraint metadata request APIs

约束主要围绕`@Constraint`注解和`javax.validation.ConstraintValidator`接口展开。

# @Constraint和ConstraintValidator

一个校验规则由一个 `javax.validation.Constraint`注解 和 一组 `javax.validation.ConstraintValidator` 接口实现类组成。

校验过程中会出现两类异常 ：

- 当Bean Validation API校验业务数据不符合时，将抛出`IllegalArgumentException `（Spring
  Web此处做了处理，将抛出`MethodArgumentNotValidException`，在**统一异常处理**
  时需要注意，参见`RequestResponseBodyMethodProcessor`）
- 其他业务无关的校验过程中出现的异常将抛出`javax.validation.ValidationException`

而观察`javax.validation.Constraint`
发现它只能注解在注解上，等于它是一个元注解，而我们放在参数上标识校验的注解需要自行编码提供。javax提供了几个常用的如：`@Email` `@Max` `@Min` `@NotEmpty` `@NotNull`
，而更复杂的业务校验注解通过自定义注解实现，如下：

```java
// 自定义ConstraintValidator
@Constraint(validatedBy = SignupUserInfoConstraintValidator.class)
// 规范强调必须支持FIELD METHOD TYPE ANNOTAION_TYPE,其他的不做要求，是否支持得看具体实现
@Target({ElementType.METHOD,ElementType.TYPE,ElementType.FIELD,ElementType.PARAMETER})
// 必须为RUNTIME
@Retention(RetentionPolicy.RUNTIME)
public @interface SignupUserInfoConstraint {
	// 添加自定义字段
    boolean enabled() default false;

    // ***************************************
    //  message, groups and payload 三个必要字段
    // ***************************************
    // 错误提示信息message建议写成"全类名.message" 主要是为了i18n
    String message() default "{com.example.validation.annotation.SignupUserInfoConstraint.message}";
    // 默认必须为空
    // groups通常用来控制constraint执行顺序，或者对javabean分组做校验
    Class<?>[] groups() default {};
    // 默认必须为空
    // payload通常用来关联元信息，比如用内部类标识，用Class而不用string主要是为了易用和类型安全
    Class<? extends Payload>[] payload() default {};
}
```

关于更多示例，请参考规范原文。

校验参数的标记由自定义注解完成，而校验逻辑需要自定义`javax.validation.ConstraintValidator` 接口实现类：

```java
// 校验注册用户信息
public class SignupUserInfoConstraintValidator implements ConstraintValidator<SignupUserInfoConstraint, UserInfo> {

    private boolean enabled = false;

    @Override
    public void initialize(SignupUserInfoConstraint constraintAnnotation) {
        this.enabled = constraintAnnotation.enabled();
    }
    
    @Override
    public boolean isValid(UserInfo userInfo, ConstraintValidatorContext context) {
        ...
    }
}
```

注意：关于该接口的泛型建议使用 *raw type*

## Graph Validation & Groups

### Graph Validation

> Consider the situation where bean X contains a field of type Y. By annotating field Y with the @Valid annotation, the
> Validator will validate Y (and its properties) when X is validated.
> The @Valid annotation is applied recursively.

案例如以上的`SignupBody`中的`SignupSource`字段

另外对于所有注解了@Valid的*Collection-valued*, *array-valued* and  *Iterable fields and properties*会校验其内容（但Map的key不会被校验）

### Groups

> During the validation call, one or more groups are validated. All the constraints belonging to this set of group is
> evaluated on the object graph.

`@Valid`注解 在*Spring Validation*的实现上不支持分组；需要用*Spring Validation*自己的`@Validated`
注解，或自定义Valid开头的注解并添加`Class[] value()`字段

**注意**：这里填的Class类型只能是**接口的Class**

# Spring 的支持

以上的场景是基于web的，实际上不管是*Jsr 303*规范本身的目的还是*Spring Validation*中提供的*Jsr 303*
支持都不仅仅可以做web数据校验，像Swing或Ui层等都可以进行校验。*Spring Validation*更像是一种适配Spring体系的领域建模。

关于*Spring Validation*的关系，我们可以参看 *Spring Validation*提供的包说明：

| package                                       | package-info                                                                                                                                                                                                                                                                                                                                                  |
|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| org.springframework.validation                | Provides **data binding** and **validation** functionality, for usage in business and/or UI layers.                                                                                                                                                                                                                                                           |
| org.springframework.validation.annotation     | Support classes for annotation-based constraint evaluation, e.g. using a JSR-303 Bean Validation provider. Provides an extended variant of JSR-303's `@Valid` supporting the specification of validation groups.                                                                                                                                              |
| org.springframework.validation.beanvalidation | Support classes for integrating a JSR-303 Bean Validation provider (such as Hibernate Validator) into a Spring ApplicationContext and in particular with Spring's data binding and validation APIs. The central class is `LocalValidatorFactoryBean` which defines a shared `ValidatorFactory`/`Validator` setup for availability to other Spring components. |
| org.springframework.validation.support        | Support classes for handling validation results.                                                                                                                                                                                                                                                                                                              |

可以看到，*Spring Validation*的目的有两个：

1. data binding ：数据绑定。如，web中将请求体内容绑定到@RequestBody实体类上
2. validation：适配*Jsr 303*，并做了扩展。

以下源码流程以*Spring Boot*+*Spring Web*为例，*Spring Web*利用*Spring Validation*来校验请求体和响应体中的内容，即*Spring
Web*提供对`@RequestBody`和`@ResponseBody`注解的参数的校验，且校验过程兼容*Jsr303*，至于*Jsr303*的实现是什么，*Spring Web*
不在乎，像这里我引入`spring-boot-starter-validation`，里面*Jsr303*的实现就是`hibernate-validator`。

## *Spring Web*中的RequestResponseBodyMethodProcessor

该类用于解析`@RequestBody`的方法参数和`@ResponseBody`的返回值，利用`HttpMessageConverter`
将请求体和响应体内容解析并生成对应的DTO和VO，在解析过程中会利用*Spring Validation*来完成校验。

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
	...
	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		...
		// web数据绑定类，即将请求体内容绑定到@RequestBody注解的实体类上
		// WebDataBinder父类DataBinder中组合了List<org.springframework.validation.Validator>
		WebDataBinder binder = binderFactory.createBinder(...);
		// 父抽象类AbstractMessageConverterMethodArgumentResolver中的方法，这里直接卸载该类中方便查看
		// 用binder进行校验并完成数据绑定
		validateIfApplicable(binder, parameter);
		if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
			// 捕获异常，抛出MethodArgumentNotValidException
			throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
		}
	}
	// 父抽象类AbstractMessageConverterMethodArgumentResolver的方法
	protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
		Annotation[] annotations = parameter.getParameterAnnotations();
		for (Annotation ann : annotations) {
			// 利用org.springframework.validation.annotation.ValidationAnnotationUtils判断是否进行校验
			// validationHints是指定的groups，必须为接口的class
			// ValidationAnnotationUtils.determineValidationHints方法如果是@Valid直接返回空数组，所以@Valid不支持分组
			// 而自定义Valid开头注解，该方法提供处理，是可以的
			Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
			if (validationHints != null) {
				// 利用WebDataBinder中的List<org.springframework.validation.Validator>进行校验
				binder.validate(validationHints);
				break;
			}
		}
	}
	...
}

```

## *Spring Validation*中的ValidationAnnotationUtils

该类属于*Spring Validation*，不是*Spring Web*的。Spring 5.3.7版本后才有。

```java
public abstract class ValidationAnnotationUtils {
	...
	/**
	 * 以下情况需要校验：
	 * 1. jsr303规范的javax.validation.Valid注解需要
	 * 2. Spring的org.springframework.validation.annotation.Validated注解需要
	 * 3. 自定义以"Valid"开头的注解需要
	 */
	public static Object[] determineValidationHints(Annotation ann) {
		Class<? extends Annotation> annotationType = ann.annotationType();
		String annotationName = annotationType.getName();
		// 1. 
		if ("javax.validation.Valid".equals(annotationName)) {
			// 直接返会空数组，不支持分组
			return EMPTY_OBJECT_ARRAY;
		}
		// 2.
		Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
		if (validatedAnn != null) {
			Object hints = validatedAnn.value();
			// 支持分组
			return convertValidationHints(hints);
		}
		// 3.
		if (annotationType.getSimpleName().startsWith("Valid")) {
			Object hints = AnnotationUtils.getValue(ann);
			// 支持分组
			return convertValidationHints(hints);
		}
		return null;
	}
	...
}
```

## *Spring Web*中的DataBinder

```java
public class DataBinder implements PropertyEditorRegistry, TypeConverter {

	...
	public void validate(Object... validationHints) {
		// 获取被校验对象，如SignupBody
		Object target = getTarget();
		// 绑定的结果
		BindingResult bindingResult = getBindingResult();
		// 开始校验
		// getValidators用SpringBooot的话，只有一个ValidatorAdapter implements SmartValidator
		for (Validator validator : getValidators()) {
			// SmartValidator是Validator的扩展，也是spring的，adding support for validation 'hints'
			// 但是这里如果是"javax.validation.Valid"注解的hints是空，所以走下面的else
			// 这里实现了jsr 303的@Valid 和 spring的@Validated注解处理的分流
			if (!ObjectUtils.isEmpty(validationHints) && validator instanceof SmartValidator) {
				((SmartValidator) validator).validate(target, bindingResult, validationHints);
			}
			else if (validator != null) {
				// ValidatorAdaptor#validate -> SpringValidator#validate -> javax.validation.Validator#validate  
				// javax.validation.Validator#validate默认实现类org.hibernate.validator.internal.engine.ValidatorImpl#validate返回一系列ConstraintViolation
				// ConstraintViolation代表着校验结果
				validator.validate(target, bindingResult);
			}
		}
	}
	...


}
```

# Hibernate的实现配置

如果不通过`springboot-starter-web`包来使用hibernate，可以直接引入下面的hibernate包的maven坐标：

```xml

<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>...</version>
</dependency>
```

并在bean配置类中设置如下配置：

```java
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;

import org.hibernate.validator.HibernateValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ValidatorConfiguration {
    @Bean
    public Validator validator() {
        ValidatorFactory validatorFactory = Validation.byProvider(HibernateValidator.class)
                .configure()
                .addProperty("hibernate.validator.fail_fast", "true")
                .buildValidatorFactory();
        return validatorFactory.getValidator();
    }
}
```

# 注解支持

目前Spring仅默认支持@RequestBody的参数校验，可以自定义AOP切面实现普通方法的校验。

注解：

```java
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 方法入参基础校验，支持{@link javax.validation.constraints}下注解 <br/>
 * {@link MethodParamBasedValidateAspect}
 * @author: baotingyu
 * @date: 2023/11/21
 **/
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface ValidateMethodArgs {

    // 分组校验，参数必须为接口Class
    Class[] groups() default {};

}

```

切面处理：

```java

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;
import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.constraints.NotNull;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.hibernate.validator.messageinterpolation.ParameterMessageInterpolator;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

/**
 * 支持方法入参校验
 * @author: baotingyu
 * @date: 2023/11/21
 **/
@Aspect
@Component
@Slf4j
// 一般在获取分布式锁&事务切面执行之前
@Order(Ordered.LOWEST_PRECEDENCE - 200)
@SuppressWarnings("rawtypes")
public class MethodParamBasedValidateAspect {

    private static final Validator validator =
            Validation.byDefaultProvider()
                    .configure()
                    .messageInterpolator(new ParameterMessageInterpolator())
                    .buildValidatorFactory()
                    .getValidator();

    @Around("@annotation(validateMethodArgs)")
    public Object around(ProceedingJoinPoint joinPoint, ValidateMethodArgs validateMethodArgs) throws Throwable {
        if (joinPoint.getArgs().length == 0) {
            return joinPoint.proceed();
        }
        try {
            Object[] args = joinPoint.getArgs();
            MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            Method method = signature.getMethod();
            Parameter[] parameters = method.getParameters();

            Set<String> nullMsg = new HashSet<>();
            Set<ConstraintViolation> allViolations = new HashSet<>();

            if (validateMethodArgs.groups().length == 0) {

                for (int i = 0; i < args.length; i++) {
                    Object arg = args[i];
                    if (Objects.isNull(arg)) {
                        if (parameters[i].isAnnotationPresent(NotNull.class)) {
                            NotNull annotation = parameters[i].getAnnotation(NotNull.class);
                            nullMsg.add(annotation.message());
                        }
                        continue;
                    }
                    allViolations.addAll(validator.validate(arg));
                }

            } else {
                // 分组校验
                for (int i = 0; i < args.length; i++) {
                    Object arg = args[i];
                    if (Objects.isNull(arg)) {
                        if (parameters[i].isAnnotationPresent(NotNull.class)) {
                            NotNull annotation = parameters[i].getAnnotation(NotNull.class);
                            nullMsg.add(annotation.message());
                        }
                        continue;
                    }
                    allViolations.addAll(validator.validate(arg, validateMethodArgs.groups()));
                }
            }

            if (nullMsg.isEmpty() && allViolations.isEmpty()) {
                return joinPoint.proceed();
            }

            StringBuilder sb = new StringBuilder();
            for (String s : nullMsg) {
                sb.append(s).append(";");
            }
            for (ConstraintViolation v : allViolations) {
                sb.append(v.getMessage()).append(";");
            }
            throw new IllegalArgumentException(sb.toString());
        } catch (Exception e) {
            log.error(e.getMessage());
            throw e;
        }
    }



}

```
