# 开发项目规范

> 项目开发中总结的开发规范，请遵守！

## 类

### 工具类   

1. 工具类默认供调用的方法都是静态方法

2. 默认私有化构造方法，不允许外部new

3. 包名默认为util，包下工具类默认为utils结尾

4. 对于诸如序列话工具类，请保持全局序列化和工具类序列化的一致性，比较推荐的做法是，项目的对象也读取自工具类中，因为静态成员要在SpringBoot初始化之前发生

5. 推荐使用单例模式管理成员，非强制

#### 工具类举例

```java

package com.phimait.lsinformatization.common.util;

import cn.hutool.core.lang.func.Func0;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.phimait.lsinformatization.common.util.serialize.BigDecimalSerializer;
import lombok.extern.slf4j.Slf4j;
import org.jetbrains.annotations.NotNull;

import java.math.BigDecimal;
import java.text.SimpleDateFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.TimeZone;

import static com.phimait.lsinformatization.common.constant.BizConstant.*;

/**
 * jackson序列化工具
 *
 * @author bwensun
 * @date 2022/3/1
 */
@Slf4j
@SuppressWarnings("unused")
public class JacksonUtils {

    private JacksonUtils(ObjectMapper objectMapper) {
    }

    public static class SingleHolder {
        private static final ObjectMapper objectMapper = new ObjectMapper();
    }

    public static ObjectMapper getObjectMapper() {
        ObjectMapper objectMapper = SingleHolder.objectMapper;
        //默认Long转String
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addSerializer(Long.class, ToStringSerializer.instance);
        simpleModule.addSerializer(Long.TYPE, ToStringSerializer.instance);
        objectMapper.registerModule(simpleModule);
        //处理bigdecimal类型
        simpleModule.addSerializer(BigDecimal.class, new BigDecimalSerializer());
        //处理java8时间类型
        final JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DATE_PATTERN)));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(NORM_DATE_PATTERN)));
        objectMapper.registerModule(javaTimeModule);
        //设置默认全局日期格式和时区
        objectMapper.setDateFormat(new SimpleDateFormat(DATE_PATTERN));
        objectMapper.setTimeZone(TimeZone.getTimeZone(TIME_ZONE));
        return objectMapper;
    }

    /**
     * 对象转Json
     *
     * @param o 对象
     * @return json字符串
     */
    public static String toJson(@NotNull Object o) {
        return doOrElse(
                () -> getObjectMapper().writeValueAsString(o), "");
    }

    /**
     * 通过class转具体对象
     * <p>一般用作单层对象转换<p>
     *
     * @param json  json字符串
     * @param clazz 泛型class
     * @param <R>   返回值
     * @return 指定类型的对象
     */
    public static <R> R fromJson(@NotNull String json, @NotNull Class<R> clazz) {
        return doOrElse(() -> getObjectMapper().readValue(json, clazz), null);
    }

    /**
     * 通过typeReference转为具体的对象
     * <p>一般用作嵌套对象的转换 比如List<UserPO> Map<UserPO> <p>
     *
     * @param json          json字符串
     * @param typeReference 类型对象
     * @param <R>           返回值
     * @return 指定类型的对象
     */
    public static <R> R fromJson(@NotNull String json, @NotNull TypeReference<R> typeReference) {
        typeReference.getType().getTypeName();
        return doOrElse(() -> getObjectMapper().readValue(json, typeReference), null);
    }

    private static <R> R doOrElse(Func0<R> supplier, R defaultResult) {
        try {
            return supplier.call();
        } catch (Exception exception) {
            log.error("【JacksonUtil】序列化失败,返回默认值", exception);
        }
        return defaultResult;
    }
}

```

### 组件类

1. 除非有必要注册为SpringBoot组件，比如需要获取SpringBoot上下文的对象，或者和SpringBoot耦合严重，否则不推荐注册为组件

2. 组件推荐使用component包名，包下类没必要一致，各自命名即可

#### 组建类举例

```java

package com.phimait.lsinformatization.common.component;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.HashOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Component;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.TimeUnit;

/**
 * 描述
 *
 * @author bwensun
 * @date 2022/5/13
 */
@Component
@SuppressWarnings({"rawtypes", "unchecked", "unused", "ConstantConditions"})
public class RedisCache {

    /**
     * 对象序列化模板
     */
    final RedisTemplate redisTemplate;

    /**
     * 字符串序列化模板
     */
    final StringRedisTemplate stringRedisTemplate;

    @Autowired
    public RedisCache(RedisTemplate redisTemplate, StringRedisTemplate stringRedisTemplate){
        this.redisTemplate = redisTemplate;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    /**
     * 存入指定值
     * 对象、string
     *
     * @param key   缓存的键值
     * @param value 缓存的值
     */
    public <T> void setCacheObject(final String key, final T value) {
        redisTemplate.opsForValue().set(key, value);
    }

}


```

--- 

## 数据库

### 建表

1. 建表表名尽量见名知意,维持可阅读的情况下尽可能短写，但不可过于缩略，比如`sys`都知道是系统，但是`enter`不一定知道是企业

2. 系统相关的表以sys开头，MySQL小写，Oracle大写

3. 表添加注释，表中字段均要加上注释，对于状态字段（存在多个固定值值）以`*_state`结尾，数据类型为长度1的数字（MySQL中为tinyInt,Oracle中为长度为1的number）各个状态对应什么含义写入注释中，对于布尔值字段以`*_flag`结尾，数据类型varchar/varchar2,长度1，`Y-true`，`N-false`，同样需要注释写入字段中

4. 表中一般不允许为null,这可能会让索引失效

5. 所有不允许为null的字段都要有默认值

6. 字段设置合适的长度，不要都走默认的255
