---
title: Jackson Json序列化反序列化的两个坑
categories: [ 编程, Java ]
tags: [ jackson ]
---

> Jackson is a suite of data-processing tools for Java (and the JVM platform)

Jackson最常用的Json序列化功能，引入如下的包即可：
```xml
<properties>
    ...
    <!-- Use the latest version whenever possible. -->
    <jackson.version>2.17.1</jackson.version>
    ...
</properties>

<dependencies>
    ...
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson.version}</version>
    </dependency>
    ...
</dependencies>
```
## Json序列化的坑

首先来看一个Json序列化异常。
`JsonUtil`工具类如下：
```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.*;

public class JsonUtil {

    private static final ObjectMapper OBJECT_MAPPER = buildLowerCamelCase();

    private static ObjectMapper buildLowerCamelCase() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        objectMapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
        objectMapper.setPropertyNamingStrategy(PropertyNamingStrategies.LOWER_CAMEL_CASE);
        return objectMapper;
    }

    public static String json(Object obj) {
        try {
            return OBJECT_MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

}
```
另外有一个实体类`CodeMonkey`:
```java
import lombok.Data;
import java.util.Objects;

@Data
public class CodeMonkey {

    private Long id;

    private String name;

    public boolean isJavaer () {
        // mock throw exception
        throw new IllegalArgumentException("should be javaer");
    }
}
```
我们对改实体对象进行Json序列化：
```java
public class JacksonTestApplication {

    public static void main(String[] args) {

        CodeMonkey codeMonkey = new CodeMonkey();
        codeMonkey.setId(1L);
        codeMonkey.setName("not bty");

        log.info("hello , {}", JsonUtil.json(codeMonkey));
    }

}
```
运行发现报错：
```java
Exception in thread "main" java.lang.RuntimeException: com.fasterxml.jackson.databind.JsonMappingException: should be javaer (through reference chain: io.github.bty834.jacksontest.CodeMonkey["javaer"])
	at io.github.bty834.jacksontest.util.JsonUtil.json(JsonUtil.java:22)
	at io.github.bty834.jacksontest.JacksonTestApplication.main(JacksonTestApplication.java:16)
Caused by: com.fasterxml.jackson.databind.JsonMappingException: should be javaer (through reference chain: io.github.bty834.jacksontest.CodeMonkey["javaer"])
	at com.fasterxml.jackson.databind.JsonMappingException.wrapWithPath(JsonMappingException.java:392)
	at com.fasterxml.jackson.databind.JsonMappingException.wrapWithPath(JsonMappingException.java:351)
	...
```

可以看到Jackson将`isJavaer`方法当作字段来序列化，但`isJavaer`其实是不需要序列化的，
可以在方法上添加 `com.fasterxml.jackson.annotation.JsonIgnore` 注解忽略该方法。


经过调试，Jackson使用`com.fasterxml.jackson.databind.introspect.POJOPropertiesCollector#collectAll`方法获取某个类型需要
序列化的属性信息: 

```java
public class POJOPropertiesCollector {
    ...
    protected void collectAll() {
        LinkedHashMap<String, POJOPropertyBuilder> props = new LinkedHashMap<String, POJOPropertyBuilder>();

          
        // 最终调用 java.lang.Class#getDeclaredFields 获取字段
        // final字段和transient字段可配置，默认都序列化
        _addFields(props); 
        // 最终调用 java.lang.Class#getDeclaredMethods 
        // 获取get/set/is开头的方法（且方法入参分别为0或1，any getter这里不讨论）
        _addMethods(props);
        ...
        // 去掉不需要的属性， 如isJavaer方法产生的javaer prop
        _removeUnwantedProperties(props);
        ...
    }
    ...
}
```

## Json反序列化的坑

回到CodeMonkey类：
```java
@Data
public class CodeMonkey {

    private Long id;

    private String name;

    private Language language;

    @JsonIgnore
    public boolean isJavaer () {
        if (Objects.equals(this.name, "bty")) {
            return true;
        }
        throw new IllegalArgumentException("should be javaer");
    }
}

@Getter
@AllArgsConstructor
public enum Language {

    JAVA(1, "Java"),
    C_PLUS_PLUS(2, "C++"),
    GO(10, "Go");

    private final int code;
    private final String name;
}
```

我们选取两个json字符串模拟反序列化：

```java
@Slf4j
public class JacksonTestApplication {
    public static void main(String[] args) {
        // language取1
        String jsonStr1 = "{\"id\":1,\"name\":\"bty\",\"language\": 1}";
        CodeMonkey codeMonkey1 = JsonUtil.fromJson(jsonStr1, CodeMonkey.class);
        System.out.println(codeMonkey1); // CodeMonkey(id=1, name=bty, language=C_PLUS_PLUS)

        // language取JAVA
        String jsonStr2 = "{\"id\":1,\"name\":\"bty\",\"language\": \"JAVA\"}";
        CodeMonkey codeMonkey2 = JsonUtil.fromJson(jsonStr2, CodeMonkey.class);
        System.out.println(codeMonkey2); // CodeMonkey(id=1, name=bty, language=JAVA)
    }
}
```

为啥我们`language`传1时，反序列化为`C_PLUS_PLUS`枚举了呢？`C_PLUS_PLUS`的`code`字段也不是1啊

在不添加jackson额外注解时，枚举序列化后是枚举值即大写英文字母，而反序列化会经过多个流程。

具体来说，枚举的反序列化默认通过`com.fasterxml.jackson.databind.deser.std.EnumDeserializer#deserialize`完成：

```java
public class EnumDeserializer
    extends StdScalarDeserializer<Object>
    implements ContextualDeserializer
{
    ...
    @Override
    public Object deserialize(JsonParser p, DeserializationContext ctxt) throws IOException
    {
        
        if (p.hasToken(JsonToken.VALUE_STRING)) {
            // 1. 如果传的是字符串，则解析字符串
            return _fromString(p, ctxt, p.getText());
        }

        // 如果是int能表示的数字（包含能转为int的long类型）
        if (p.hasToken(JsonToken.VALUE_NUMBER_INT)) { 
            
            if (_isFromIntValue) {  // 判断根据@JsonValue的值来解析

                // 将数字转换为字符串根据@JsonValue注解的字段或方法解析
                return _fromString(p, ctxt, p.getText());
            }
            
            // 根据反序列化配置的CoercionAction取值或报错
            // CoercionAction 分为 Fail , TryConvert , AsNull , AsEmpty
            // 枚举这里为TryConvert，具体逻辑参考：com.fasterxml.jackson.databind.cfg.CoercionConfigs#findCoercion
            // TryConvert会使用枚举值在枚举中的index来匹配这里的int，很坑！
            return _fromInteger(p, ctxt, p.getIntValue());
        }
        
        // 解析嵌套json对象
        if (p.isExpectedStartObjectToken()) {
            return _fromString(p, ctxt,
                    ctxt.extractScalarFromObject(p, this, _valueClass));
        }
        // 解析嵌套json数组
        return _deserializeOther(p, ctxt);
    }
    ...
}
```

如果规避以上的坑呢？

### @JsonValue
`code`字段添加`@JsonValue`，则反序列化只能根据code值(int或int的字符串都行)进行，除了会影响反序列化，序列化时会将该枚举写成其`code`值
```java
@Getter
@AllArgsConstructor
public enum Language {

    JAVA(1L, "Java"),
    C_PLUS_PLUS(2L, "C++"),
    GO(10L, "Go");

    @JsonValue
    private final long code;
    private final String name;
}
```

### @JsonCreator
2. 如果我既想要根据`code`字段也想要根据`Enum.name()`来反序列化呢？自定义反序列化方法并注解`@JsonCreator`

```java
@Getter
@AllArgsConstructor
public enum Language {

    JAVA(1L, "Java"),
    C_PLUS_PLUS(2L, "C++"),
    GO(10L, "Go");

    @JsonValue
    private final long code;
    private final String name;

    @JsonCreator
    public static Language parse(String val) {
        for (Language value : Language.values()) {
            // 先根据Enum.name()判断
            if (value.name().equals(val)) {
                return value;
            }
            // 再根据code字段判断
            if (Objects.equals(String.valueOf(value.code), val)) {
                return value;
            }
        }
        return null;
    }
}
```
### DeserializationFeature.FAIL_ON_NUMBERS_FOR_ENUMS

可以打开反序列化配置`objectMapper.enable(DeserializationFeature.FAIL_ON_NUMBERS_FOR_ENUMS);` 来避免int匹配不到时根据index来匹配，会直接报错。但并不影响`@JsonValue`和`@JsonCreator`的使用。
