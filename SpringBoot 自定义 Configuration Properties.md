@ConfigurationProperties 加载配置是通过 BeanPostProcessor 实现，其对应的Bean的后置处理器为 ConfigurationPropertiesBindingPostProcessor。也就是说在bean被实例化后，会调用后置处理器，递归的查找属性，通过反射机制注入值，因此属性需要提供setter和getter方法。

此外，针对此种属性注入的方式，SpringBoot支持Relaxed Binding，即只需保证配置文件的属性和setter方法名相同即可。


|**在SpringBoot官方文档中有几个注意点：**|
|:-----------------------------------|
|属性必须要有getter、setter方法。|
|如果属性的类型是集合，要确保集合是不可变的。|
|如果使用Lombok自动生成getter/setter方法，一定不要生成对应的任何构造函数，因为Spring IOC容器会自动使用它来实例化对象。|
|使用JavaBean属性绑定的方式只针对标准 Java Bean 属性，不支持对静态属性的绑定。|


当我们使用 @ConfigurationProperties 注解进行属性注入时，记得在 pom.xml 文件中添加 spring-boot-configuration-processor 依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
</dependency>
```

Spring考虑到带有 @ConfigurationProperties 注解的类可能不适合扫描（比如基于条件启动），所以 Spring 并不会自动扫描带有 @ConfigurationProperties 注解的类。

在这些情况下，推荐使用 @EnableConfigurationProperties 注解指定要处理的类型列表（即：将 @ConfigurationProperties 注释的类加到 Spring IOC 容器中）。一般通过将 @EnableConfigurationProperties 加在 @Configuration 类上完成。

这种方式在Spring源码中被广泛使用，即：

```java
@Configuration
@EnableConfigurationProperties(MonitorConfigurationProperties.class)
public class Config {

}
```

```java
@ConfigurationProperties(prefix = "monitor")
public class MonitorConfigurationProperties {

    /**
     * 应用名称
     */
    private String appId;

    /**
     * 服务地址
     */
    private String serverUrl = "localhost:8080";

    /**
     * 集合
     */
    private List<Integer> data = new ArrayList<>();

    // Getter/Setter ...
}

```

另外，也可以直接在 JavaBean 上使用 @Configuration 和 @ConfigurationProperties 注解，让Spring IOC可以自动扫描到 @ConfigurationProperties 注解标注的类（不过不推荐）：

```java
@Configuration
@ConfigurationProperties(prefix = "monitor")
public class MonitorConfigurationProperties {

    /**
     * 应用名称
     */
    private String appId;

    /**
     * 服务地址
     */
    private String serverUrl = "localhost:8080";

    /**
     * 集合
     */
    private List<Integer> data = new ArrayList<>();

    // Getter/Setter ...
}
```

# JSR-303 对 @ConfigurationProperties 验证

@ConfigurationProperties 还提供了对 JSR-303 注解验证支持，在 pom 文件中添加 spring-boot-starter-validation 依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

当你在对含有 @ConfigurationProperties 注解的 JavaBean 使用Spring 的 @Validated 注解时，Spring Boot 都会尝试验证此类：

```java
@Validated
@ConfigurationProperties(prefix = "monitor")
public class MonitorConfigurationProperties {

    /**
     * 应用名称
     */
    @NotBlank
    private String appId;

    /**
     * 服务地址
     */
    private String serverUrl = "localhost:8080";

    /**
     * 集合
     */
    private List<Integer> data = new ArrayList<>();

    // Getter/Setter ...
}
```

<img width="600px" src="http://spring-media.knowledge.ituknown.cn/ConfigurationProperties/configuration-properties-validated.png" alt="configuration-properties-validated.png">

<img width="600px" src="http://spring-media.knowledge.ituknown.cn/ConfigurationProperties/configuration-properties-metadata-json.png" alt="configuration-properties-metadata-json.pngg">

<img width="600px" src="http://spring-media.knowledge.ituknown.cn/ConfigurationProperties/configuration-properties-tips.png" alt="configuration-properties-tips.png">

--

https://docs.spring.io/spring-boot/docs/current/reference/html/configuration-metadata.html

https://docs.spring.io/spring-boot/docs/2.1.17.RELEASE/reference/html/configuration-metadata.html#configuration-metadata-annotation-processor

https://www.mdoninger.de/2015/05/16/completion-for-custom-properties-in-spring-boot.html

https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/boot-features-external-config.html#boot-features-external-config-typesafe-configuration-properties

https://docs.spring.io/spring-boot/docs/2.7.15/reference/html/features.html#features.external-config.typesafe-configuration-properties.java-bean-binding