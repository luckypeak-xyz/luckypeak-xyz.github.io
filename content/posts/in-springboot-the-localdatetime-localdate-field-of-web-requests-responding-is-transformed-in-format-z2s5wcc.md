---
title: SpringBoot 中对 Web 请求\响应 的 LocalDateTime\LocalDate 字段进行格式转换
slug: >-
  in-springboot-the-localdatetime-localdate-field-of-web-requests-responding-is-transformed-in-format-z2s5wcc
url: >-
  /post/in-springboot-the-localdatetime-localdate-field-of-web-requests-responding-is-transformed-in-format-z2s5wcc.html
date: '2025-01-02 22:07:18+08:00'
lastmod: '2025-01-05 21:16:04+08:00'
toc: true
isCJKLanguage: true
---



## 默认序列化格式

我们先来看看默认情况下时间类型是如何序列的。测试代码如下：

```java
@Slf4j
@RestController
@RequestMapping("/date-time-format-learning")
@RequiredArgsConstructor
public class SpringDateTimeFormatLearningController {

    private final ObjectMapper objectMapper;

    @Data
    public static class TimeTypeSet implements Serializable {
        private static final long serialVersionUID = 1L;

        private Date date;
        private LocalDate localDate;
        private LocalTime localTime;
        private LocalDateTime localDateTime;

    }

    @GetMapping("/default-serialize-format")
    public TimeTypeSet defaultSerializeFormat() {
        TimeTypeSet timeTypeSet = new TimeTypeSet();
        timeTypeSet.setDate(new Date());
        timeTypeSet.setLocalDate(LocalDate.now());
        timeTypeSet.setLocalTime(LocalTime.now());
        timeTypeSet.setLocalDateTime(LocalDateTime.now());

        try {
            String timeTypeSetJsonStr = objectMapper.writeValueAsString(timeTypeSet);
            log.info("Spring 默认的序列化格式，timeTypeSetJsonStr: {}", timeTypeSetJsonStr);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        ObjectMapper newObjectMapper = new ObjectMapper();
        try {
            String timeTypeSetJsonStr = newObjectMapper.writeValueAsString(timeTypeSet);
            log.info("Jackson 默认的序列化格式，timeTypeSetJsonStr: {}", timeTypeSetJsonStr);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        return timeTypeSet;
    }

}
```

* 定义 TimeTypeSet 类，其中包含了我们日常开发中常用的时间类型；
* defaultSerializeFormat 中我们测试了三种类型的默认序列化格式：

  * Spring 默认的序列化格式，这里通过 Spring 容器中的 ObjectMapper 来序列化 TimeTypeSet；
  * Jackson 默认的序列化格式，这里通过创建一个全新的 ObjectMapper 来序列化 TimeTypeSet；
  * Spring Web 响应的默认序列化格式，这里通过查看接口响应来查看 Spring Web 是如何序列化响应的。

调用接口后，查看 Console 输出：

```shell
s.SpringDateTimeFormatLearningController : Spring 默认的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-04T12:12:00.453+00:00","localDate":"2025-01-04","localTime":"20:12:00.453861","localDateTime":"2025-01-04T20:12:00.453876"}
o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Java 8 date/time type `java.time.LocalDate` not supported by default: add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310" to enable handling (through reference chain: xyz.luckypeak.playground.springdatetimeformatlearning.SpringDateTimeFormatLearningController$TimeTypeSet["localDate"])] with root cause
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Java 8 date/time type `java.time.LocalDate` not supported by default: add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310" to enable handling (through reference chain: xyz.luckypeak.playground.springdatetimeformatlearning.SpringDateTimeFormatLearningController$TimeTypeSet["localDate"])
```

可以看到抛异常了。为了处理 `java.time`​ 中的时间类型，需要注册 JavaTime 模块。修改上述代码：

```java
// ...
ObjectMapper newObjectMapper = new ObjectMapper();
newObjectMapper.registerModule(new JavaTimeModule());
try {
    String timeTypeSetJsonStr = newObjectMapper.writeValueAsString(timeTypeSet);
    log.info("Jackson 默认的序列化格式，timeTypeSetJsonStr: {}", timeTypeSetJsonStr);
// ...
```

再次调用接口，查看 Console 输出：

```shell
s.SpringDateTimeFormatLearningController : Spring 默认的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-04T12:33:31.172+00:00","localDate":"2025-01-04","localTime":"20:33:31.172349","localDateTime":"2025-01-04T20:33:31.172378"}
s.SpringDateTimeFormatLearningController : Jackson 默认的序列化格式，timeTypeSetJsonStr: {"date":1735994011172,"localDate":[2025,1,4],"localTime":[20,33,31,172349000],"localDateTime":[2025,1,4,20,33,31,172378000]}
```

接口响应：

```json
{
  "date": "2025-01-04T12:33:31.172+00:00",
  "localDate": "2025-01-04",
  "localTime": "20:33:31.172349",
  "localDateTime": "2025-01-04T20:33:31.172378"
}
```

可以看到，对于各种时间类型 Spring 默认的序列化格式 和 Spring Web 响应的默认序列化格式 都相同，且和 Jackson 默认的序列化格式 不同。那么，Spring 对 Jackson 做了哪些默认设置呢？

### Spring 的 Jackson 默认设置

Spring 的 Jackson 配置属性类是 `JacksonProperties`​：

```java
@ConfigurationProperties(prefix = "spring.jackson")
public class JacksonProperties {

	/**
	 * Date format string or a fully-qualified date format class name. For instance,
	 * 'yyyy-MM-dd HH:mm:ss'.
	 */
	private String dateFormat;

	/**
	 * One of the constants on Jackson's PropertyNamingStrategies. Can also be a
	 * fully-qualified class name of a PropertyNamingStrategy implementation.
	 */
	private String propertyNamingStrategy;

	/**
	 * Jackson visibility thresholds that can be used to limit which methods (and fields)
	 * are auto-detected.
	 */
	private final Map<PropertyAccessor, JsonAutoDetect.Visibility> visibility = new EnumMap<>(PropertyAccessor.class);

	/**
	 * Jackson on/off features that affect the way Java objects are serialized.
	 */
	private final Map<SerializationFeature, Boolean> serialization = new EnumMap<>(SerializationFeature.class);

	/**
	 * Jackson on/off features that affect the way Java objects are deserialized.
	 */
	private final Map<DeserializationFeature, Boolean> deserialization = new EnumMap<>(DeserializationFeature.class);

	/**
	 * Jackson general purpose on/off features.
	 */
	private final Map<MapperFeature, Boolean> mapper = new EnumMap<>(MapperFeature.class);

	/**
	 * Jackson on/off features for parsers.
	 */
	private final Map<JsonParser.Feature, Boolean> parser = new EnumMap<>(JsonParser.Feature.class);

	/**
	 * Jackson on/off features for generators.
	 */
	private final Map<JsonGenerator.Feature, Boolean> generator = new EnumMap<>(JsonGenerator.Feature.class);

	/**
	 * Controls the inclusion of properties during serialization. Configured with one of
	 * the values in Jackson's JsonInclude.Include enumeration.
	 */
	private JsonInclude.Include defaultPropertyInclusion;

	/**
	 * Global default setting (if any) for leniency.
	 */
	private Boolean defaultLeniency;

	/**
	 * Strategy to use to auto-detect constructor, and in particular behavior with
	 * single-argument constructors.
	 */
	private ConstructorDetectorStrategy constructorDetector;

	/**
	 * Time zone used when formatting dates. For instance, "America/Los_Angeles" or
	 * "GMT+10".
	 */
	private TimeZone timeZone = null;

	/**
	 * Locale used for formatting.
	 */
	private Locale locale;

	// ...
}
```

这么多配置属性，Spring 在创建默认的 ObjectMapper Bean 时会做哪些配置呢？我们通过在 Spring 创建默认 ObjectMapper Bean 的[代码](https://github.com/spring-projects/spring-boot/blob/f8c9fee3b0c8ff9ef48cf12fb4a9f8a51630a485/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration.java#L103)处打断点，来查看 Spring 做了哪些默认配置。结合代码可以发现，Spring 关于时间处理在序列化特性上做了如下[两个配置](https://github.com/spring-projects/spring-boot/blob/f8c9fee3b0c8ff9ef48cf12fb4a9f8a51630a485/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration.java#L85-L86)：

```java
featureDefaults.put(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
featureDefaults.put(SerializationFeature.WRITE_DURATIONS_AS_TIMESTAMPS, false);
```

我们同样修改 Jackson 的上述配置：

```java
// ...
newObjectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
newObjectMapper.configure(SerializationFeature.WRITE_DURATIONS_AS_TIMESTAMPS, false);
try {
    String timeTypeSetJsonStr = newObjectMapper.writeValueAsString(timeTypeSet);
    log.info("Jackson 自定义的序列化格式，timeTypeSetJsonStr: {}", timeTypeSetJsonStr);
} catch (JsonProcessingException e) {
    throw new RuntimeException(e);
}
// ...
```

调用接口，查看 Console 输出：

```shell
s.SpringDateTimeFormatLearningController : Spring 默认的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-05T00:24:24.314+00:00","localDate":"2025-01-05","localTime":"08:24:24.31458","localDateTime":"2025-01-05T08:24:24.314627"}
s.SpringDateTimeFormatLearningController : Jackson 默认的序列化格式，timeTypeSetJsonStr: {"date":1736036664314,"localDate":[2025,1,5],"localTime":[8,24,24,314580000],"localDateTime":[2025,1,5,8,24,24,314627000]}
s.SpringDateTimeFormatLearningController : Jackson 自定义的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-05T00:24:24.314+00:00","localDate":"2025-01-05","localTime":"08:24:24.31458","localDateTime":"2025-01-05T08:24:24.314627"}
```

可以看到，修改之后 Spring 默认的序列化格式 和 Jackson 自定义的序列化格式 也相同了。

到这里我们会想到，通过自定义配置 Spring 的 ObjectMapper 可以实现对 Web 请求\响应 的 LocalDateTime\LocalDate 字段进行格式转换。

## 自定义配置 Spring ObjectMapper 的 serializer

### 自定义 serializer

分别新建 `LocalDateTimeToTimestampMillisSerializer`​ 和 `DateToTimestampMillisSerializer`​ 两个 序列化器 完成从 LocalDateTime 和 Date 到毫秒时间戳的序列化：

```java
// LocalDateTimeToTimestampMillisSerializer.java
public class LocalDateTimeToTimestampMillisSerializer extends JsonSerializer<LocalDateTime> {
    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(Optional.ofNullable(value).map(ldt -> {
                    ZonedDateTime zdt = ZonedDateTime.of(ldt, ZoneId.systemDefault());
                    return String.valueOf(zdt.toInstant().toEpochMilli());
                }).orElse(null)
        );
    }
}

// DateToTimestampMillisSerializer.java
public class DateToTimestampMillisSerializer extends JsonSerializer<Date> {
    @Override
    public void serialize(Date value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(Optional.ofNullable(value).map(d -> String.valueOf(d.getTime())).orElse(null));
    }
}
```

### 设置自定义 serializer

#### 方法一：完全替换 Spring ObjectMapper 为自定义 ObjectMapper

新建 Jackson 配置类 JacksonConfig，注册自定义 ObjectMapper。

```java
// JacksonConfig.class

@Configuration
public class JacksonConfig {

    @Bean
    @Primary
    @ConditionalOnMissingBean(ObjectMapper.class)
    public ObjectMapper objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();

        JavaTimeModule javaTimeModule = javaTimeModule();
        objectMapper.registerModule(javaTimeModule);

        return objectMapper;
    }

    public JavaTimeModule javaTimeModule() {
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeToTimestampMillisSerializer());
        javaTimeModule.addSerializer(Date.class, new DateToTimestampMillisSerializer());
        return javaTimeModule;
    }

}
```

调用接口，查看 Console 输出：

```shell
s.SpringDateTimeFormatLearningController : Spring 默认的序列化格式，timeTypeSetJsonStr: {"date":"1736065963701","localDate":[2025,1,5],"localTime":[16,32,43,702035000],"localDateTime":"1736065963702"}
s.SpringDateTimeFormatLearningController : Jackson 默认的序列化格式，timeTypeSetJsonStr: {"date":1736065963701,"localDate":[2025,1,5],"localTime":[16,32,43,702035000],"localDateTime":[2025,1,5,16,32,43,702054000]}
s.SpringDateTimeFormatLearningController : Jackson 自定义的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-05T08:32:43.701+00:00","localDate":"2025-01-05","localTime":"16:32:43.702035","localDateTime":"2025-01-05T16:32:43.702054"}
```

查看接口响应：

```json
{
  "date": "1736065963701",
  "localDate": [
    2025,
    1,
    5
  ],
  "localTime": [
    16,
    32,
    43,
    702035000
  ],
  "localDateTime": "1736065963702"
}
```

Date 和 LocalDateTime 类型的 Spring 序列化格式 和 Spring Web 响应序列化格式都为毫秒时间戳。LocalDate 和 LocalTime 的 Spring Web 响应序列化格式则变为和 Jackson 默认的序列化格式相同的数组形式。

#### 方法二：注册 JavaTimeModule Bean

同方法一需要新建 Jackson 配置类 JacksonConfig，但是不去完全替换 Spring 的 ObjectMapper，而是注册 JavaTimeModule Bean，由 Spring 将此自定义配置加入到其 ObjectMapper 配置中。

```java
// JacksonConfig.class

@Configuration
public class JacksonConfig {

	@Bean
    private JavaTimeModule javaTimeModule() {
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeToTimestampSerializer());
        javaTimeModule.addSerializer(Date.class, new DateToTimestampSerializer());
        return javaTimeModule;
    }

}
```

调用接口，查看 Console 输出：

```shell
s.SpringDateTimeFormatLearningController : Spring 默认的序列化格式，timeTypeSetJsonStr: {"date":"1736066088087","localDate":"2025-01-05","localTime":"16:34:48.087914","localDateTime":"1736066088087"}
s.SpringDateTimeFormatLearningController : Jackson 默认的序列化格式，timeTypeSetJsonStr: {"date":1736066088087,"localDate":[2025,1,5],"localTime":[16,34,48,87914000],"localDateTime":[2025,1,5,16,34,48,87933000]}
s.SpringDateTimeFormatLearningController : Jackson 自定义的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-05T08:34:48.087+00:00","localDate":"2025-01-05","localTime":"16:34:48.087914","localDateTime":"2025-01-05T16:34:48.087933"}
```

查看接口响应：

```json
{
  "date": "1736066088087",
  "localDate": "2025-01-05",
  "localTime": "16:34:48.087914",
  "localDateTime": "1736066088087"
}
```

Date 和 LocalDateTime 类型的 Spring 序列化格式 和 Spring Web 响应序列化格式都为毫秒时间戳。LocalDate 和 LocalTime 的 Spring Web 响应序列化格式则依然保持 Spring 默认的序列化格式。

Spring 在构建其 ObjectMapper 时会通过 [configureModules](https://github.com/spring-projects/spring-boot/blob/f8c9fee3b0c8ff9ef48cf12fb4a9f8a51630a485/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration.java#L284-L286) 方法完成 JavaTimeModule 的注册：

```java
private void configureModules(Jackson2ObjectMapperBuilder builder) {
	Collection<Module> moduleBeans = getBeans(this.applicationContext, Module.class);
	builder.modulesToInstall(moduleBeans.toArray(new Module[0]));
}
```

#### 方法三：通过 @JsonComponet 注解注册自定义 serializer

新建 CustomJsonComponet 类，并新建分别继承 DateToTimestampMillisSerializer 和 LocalDateTimeToTimestampMillisSerializer 的内部类  `CustomDateToTimestampMillisSerializer`​ 和 `CustomLocalDateTimeToTimestampMillisSerializer`​，以完成自定义 serializer 注册。

```java
// CustomJsonComponent.java

@JsonComponent
public class CustomJsonComponent {
    public static class CustomDateToTimestampMillisSerializer extends DateToTimestampMillisSerializer {};
    public static class CustomLocalDateTimeToTimestampMillisSerializer extends LocalDateTimeToTimestampMillisSerializer {};
}
```

调用接口，查看 Console 输出：

```shell
s.SpringDateTimeFormatLearningController : Spring 默认的序列化格式，timeTypeSetJsonStr: {"date":"1736066156601","localDate":"2025-01-05","localTime":"16:35:56.601981","localDateTime":"1736066156601"}
s.SpringDateTimeFormatLearningController : Jackson 默认的序列化格式，timeTypeSetJsonStr: {"date":1736066156601,"localDate":[2025,1,5],"localTime":[16,35,56,601981000],"localDateTime":[2025,1,5,16,35,56,601995000]}
s.SpringDateTimeFormatLearningController : Jackson 自定义的序列化格式，timeTypeSetJsonStr: {"date":"2025-01-05T08:35:56.601+00:00","localDate":"2025-01-05","localTime":"16:35:56.601981","localDateTime":"2025-01-05T16:35:56.601995"}
```

查看接口响应：

```json
{
  "date": "1736066156601",
  "localDate": "2025-01-05",
  "localTime": "16:35:56.601981",
  "localDateTime": "1736066156601"
}
```

这里对于输出结果的分析同 方法二。

​`JacksonAutoConfiguration`​ 配置注册了 `JsonComponentModule`​ Bean：

```java
@Bean
public JsonComponentModule jsonComponentModule() {
	return new JsonComponentModule();
}
```

​`JsonComponentModule`​ 实现了 `InitializingBean`​ 接口，在 `afterPropertiesSet`​ 方法中调用 [`registerJsonComponents`](https://github.com/spring-projects/spring-boot/blob/383f1964e62e4ad5626ad8d52fe0fcdfe37152f4/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/jackson/JsonComponentModule.java#L77)​ 方法完成自定义 serializer 的注册。

## 默认反序列化格式

知道了默认的序列化输出格式，那么推测默认的反序列化的输入就只能和序列化的输出格式相同了，使用如下代码验证：

```java
@PostMapping("/default-serialize-format")
public void defaultSerializeFormat(
        @RequestBody TimeTypeSet timeTypeSet
) {
	log.info("timeTypeSet: {}", timeTypeSet);
}
```

使用如下 body 请求上述 post 接口：

```json
{
  "date": "2025-01-05T09:03:24.687+00:00",
  "localDate": "2025-01-05",
  "localTime": "17:03:24.688091",
  "localDateTime": "2025-01-05T17:03:24.68812"
}
```

没问题，代码正常执行。将 date 和 localDateTime 改为时间戳再次尝试：

```json
{
  "date": "1736066156601",
  "localDate": "2025-01-05",
  "localTime": "16:35:56.601981",
  "localDateTime": "1736066156601"
}
```

返回了 400，打印了如下日志：

```shell
.w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot deserialize value of type `java.time.LocalDateTime` from String "1736066156601": Failed to deserialize java.time.LocalDateTime: (java.time.format.DateTimeParseException) Text '1736066156601' could not be parsed at index 0; nested exception is com.fasterxml.jackson.databind.exc.InvalidFormatException: Cannot deserialize value of type `java.time.LocalDateTime` from String "1736066156601": Failed to deserialize java.time.LocalDateTime: (java.time.format.DateTimeParseException) Text '1736066156601' could not be parsed at index 0<EOL> at [Source: (org.springframework.util.StreamUtils$NonClosingInputStream); line: 5, column: 20] (through reference chain: xyz.luckypeak.playground.springdatetimeformatlearning.SpringDateTimeFormatLearningController$TimeTypeSet["localDateTime"])]
```

代码遇到了 DateTimeParseException 异常。

在日常开发中，外部输入的时间格式多种多样，ISO 8601 标准格式 和 毫秒时间戳 都有可能出现，那么如何配置 ObjectMapper 使其可以应对各种可能得时间格式呢？

## 自定义配置 Spring ObjectMapper 的 deserializer

分别新建 `CommonLocalDateTimeDeserializer`​、`CommonLocalDateDeserializer`​  和 `CommonDateDeserializer`​ 两个 反序列化器 完成从 各种可能时间格式到 LocalDateTime、LocalDate 和 Date 的反序列化：

```java
// DeserializerCommon.java

public class DeserializerCommon {

    public static final DateTimeFormatter dateTimeFormatter = new DateTimeFormatterBuilder()
            .appendOptional(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSSSS"))
            .appendOptional(DateTimeFormatter.ISO_DATE_TIME)
            .toFormatter();
    public static final DateTimeFormatter dateFormatter = new DateTimeFormatterBuilder()
            .appendOptional(DateTimeFormatter.ISO_LOCAL_DATE)
            .appendOptional(DateTimeFormatter.BASIC_ISO_DATE)
            .toFormatter();
    public static final int ASCII_COLON = ':';

    /**
     * 判断是否是毫秒时间戳
     * 大于 10_000_000_000L 则认为是毫秒时间戳
     * 10_000_000_000 / (365 * 24 * 60 * 60) ≈ 317.0979198376 + 1_970 ≈ 2_287
     * 秒时间戳大于 10_000_000_000L 还在 2287 年
     * @param ts 时间戳
     * @return 是否是毫秒时间戳
     */
    public static boolean isMillis(long ts) {
        return ts > (10_000_000_000L);
    }

    /**
     * 是否是秒时间戳
     * 小于 10_000_000_000L 且大于 100_000_000L 则认为是秒时间戳
     * 10_000_000_000 / (365 * 24 * 60 * 60) ≈ 317.0979198376 + 1_970 ≈ 2_287
     * 秒时间戳大于 10_000_000_000L 还在 2287 年
     * 100_000_000L / (365 * 24 * 60 * 60) ≈ 3.1709791984 + 1_970 ≈ 1_973
     * 秒时间戳小于 100_000_000L 则在 1973 年
     * @param ts 时间戳
     * @return 是否是秒时间戳
     */
    public static boolean isSeconds(long ts) {
        return ts <= (10_000_000_000L) && ts > 100_000_000L;
    }

    /**
     * 是否是日期字符串
     * 如果字符串中不包含冒号 ({@code :})，则认为是日期字符串
     * @param s 字符串
     * @return 是否是日期字符串
     */
    public static boolean isDate(@NonNull String s) {
        return s.indexOf(ASCII_COLON) == -1;
    }

}
```

```java
// CommonLocalDateTimeDeserializer.java

public class CommonLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {
    @Override
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JacksonException {
        if (StrUtil.isBlank(p.getText())) {
            return null;
        }

        if (StrUtil.isNumeric(p.getText())) {
            long ts = NumberUtil.parseLong(p.getText());
            if (DeserializerCommon.isMillis(ts)) {
                return LocalDateTime.ofInstant(Instant.ofEpochMilli(ts), ZoneId.systemDefault());
            } else if (DeserializerCommon.isSeconds(ts)) {
                return LocalDateTime.ofInstant(Instant.ofEpochSecond(ts), ZoneId.systemDefault());
            }
        }

        if (DeserializerCommon.isDate(p.getText())) {
            LocalDate ld = LocalDate.parse(p.getText(), DeserializerCommon.dateFormatter);
            return ld.atStartOfDay();
        }

        try {
            ZonedDateTime zdt = ZonedDateTime.parse(p.getText(), DeserializerCommon.dateTimeFormatter);
            return zdt.withZoneSameInstant(ZoneId.systemDefault()).toLocalDateTime();
        } catch (DateTimeParseException e) {
            return LocalDateTime.parse(p.getText(), DeserializerCommon.dateTimeFormatter);
        }
    }
}
```

```java
// CommonLocalDateDeserializer.java

public class CommonLocalDateDeserializer extends JsonDeserializer<LocalDate> {

    @Override
    public LocalDate deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JacksonException {
        if (StrUtil.isBlank(p.getText())) {
            return null;
        }

        return LocalDate.parse(p.getText(), DeserializerCommon.dateFormatter);
    }
}
```

```java
// CommonDateDeserializer.java

public class CommonDateDeserializer extends JsonDeserializer<Date> {

    @Override
    public Date deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JacksonException {
        if (StrUtil.isBlank(p.getText())) {
            return null;
        }

        if (StrUtil.isNumeric(p.getText())) {
            long ts = NumberUtil.parseLong(p.getText());
            if (DeserializerCommon.isMillis(ts)) {
                return new Date(ts);
            } else if (DeserializerCommon.isSeconds(ts)) {
                return new Date(ts * 1000L);
            }
        }

        if (DeserializerCommon.isDate(p.getText())) {
            LocalDate localDate = LocalDate.parse(p.getText(), DeserializerCommon.dateFormatter);
            LocalDateTime localDateTime = localDate.atStartOfDay();
            ZoneId zoneId = ZoneId.systemDefault();
            Instant instant = localDateTime.atZone(zoneId).toInstant();
            return Date.from(instant);
        }

        try {
            ZonedDateTime zdt = ZonedDateTime.parse(p.getText(), DeserializerCommon.dateTimeFormatter);
            Instant instant = zdt.toInstant();
            return Date.from(instant);
        } catch (DateTimeParseException e) {
            LocalDateTime ldt = LocalDateTime.parse(p.getText(), DeserializerCommon.dateTimeFormatter);
            ZoneId zoneId = ZoneId.systemDefault();
            Instant instant = ldt.atZone(zoneId).toInstant();
            return Date.from(instant);
        }
    }
}
```

设置自定义 deserializer 的方法同 serializer。设置完成后，再次验证：

请求体：

```json
{
  "date": "2025-01-05T09:03:24.687+00:00",
  "localDate": "2025-01-05",
  "localTime": "17:03:24.688091",
  "localDateTime": "2025-01-05T17:03:24.68812"
}
```

Console 输出：

```shell
s.SpringDateTimeFormatLearningController : timeTypeSet: SpringDateTimeFormatLearningController.TimeTypeSet(date=Sun Jan 05 17:03:24 CST 2025, localDate=2025-01-05, localTime=17:03:24.688091, localDateTime=2025-01-05T17:03:24.688120)
```

请求体：

```json
{
  "date": "1736066156601",
  "localDate": "2025-01-05",
  "localTime": "16:35:56.601981",
  "localDateTime": "1736066156601"
}
```

Console 输出：

```json
s.SpringDateTimeFormatLearningController : timeTypeSet: SpringDateTimeFormatLearningController.TimeTypeSet(date=Sun Jan 05 16:35:56 CST 2025, localDate=2025-01-05, localTime=16:35:56.601981, localDateTime=2025-01-05T16:35:56.601)
```

请求体：

```json
{
  "date": "2025-01-05T09:03:24.687+00:00",
  "localDate": "20250105",
  "localTime": "16:35:56.601981",
  "localDateTime": "1736066156601"
}
```

Console 输出：

```shell
s.SpringDateTimeFormatLearningController : timeTypeSet: SpringDateTimeFormatLearningController.TimeTypeSet(date=Sun Jan 05 17:03:24 CST 2025, localDate=2025-01-05, localTime=16:35:56.601981, localDateTime=2025-01-05T16:35:56.601)
```

请求体：

```json
{
  "date": "1736066156601",
  "localDate": "2025-01-05",
  "localTime": "16:35:56.601981",
  "localDateTime": "20250105"
}
```

Console 输出：

```shell
s.SpringDateTimeFormatLearningController : timeTypeSet: SpringDateTimeFormatLearningController.TimeTypeSet(date=Sun Jan 05 16:35:56 CST 2025, localDate=2025-01-05, localTime=16:35:56.601981, localDateTime=2025-01-05T00:00)
```

可以看到现在 deserializer 可以应对时间类型的多种不同格式了。

有个需要注意的地方是，如果时间戳小于 100_000_000L 或者大于 10_000_000_000L 会有问题。
