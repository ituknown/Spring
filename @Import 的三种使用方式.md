# 前言

`@Import` 注解是 Spring 从 3.0 开始提供的具有导入一个或多个通用组件的注解。在 Spring 源码中随处可见，尤其是在元注解上更是常用。比如 Spring 的声明式事务注解，在其注解上就使用了 `@Import` 注解：


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)  // <===== 看下这里
public @interface EnableTransactionManagement {
    // ...
}
```

另外，在 SpringBoot 以及 SpringCloud 中我们都知道其原理是 **自动装配** 。而自动装配的原理的底层原理其实就是基于该注解实现的，如果你没听过这个词那你是否使用过 `@Enable` 开头的注解呢？

比如 `@SpringBootApplication` 注解！是不是很熟悉？没错这个注解就是常常出现在启动类上的那个注解。来看下该注解源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration     // <===== 看下这里
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
public @interface SpringBootApplication {
    // ...
}

```

在该注解之上使用了 `@EnableAutoConfiguration` 注解，现在再来看下该注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)  // <===== 看下这里
public @interface EnableAutoConfiguration {
    // ...
}
```

如果你不信我们在开看下 SpringCloud 的服务注册发现的注解： `@EnableDiscoveryClient` ：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)  // <===== 看下这里
public @interface EnableDiscoveryClient {
    // ...
}
```

看了这么多，有没有体会到 `@Import` 注解的魅力呢？现在我们具体来看下该注解！

在 Spring 源码中，对 `@Import` 的解释是：

Indicates one or more *component classes* to import; typically `@Configuration` classes.

Provides functionality equivalent to the `<import/>`  element in Spring XML. Allows for importing `@Configuration`  classes, `ImportSelector.class`  and `ImportBeanDefinitionRegistrar.class` implementations, as well as regular component classes.

总结下来就是： `@Import` 注解与基于 XML 配置的 `<import />` 具有等效作；可以用来导入普通类、基于 `@Configuration` 注解的配置类、 `org.springframework.context.annotation.ImportSelector` 的实现类和 `org.springframework.context.annotation.ImportBeanDefinitionRegistrar` 的实现类。即：

| **@Import 应用**                                   |
| :------------------------------------------------- |
| 导入普通类                                         |
| 导入基于 `@Configuration` 注解的 JavaConfig 配置类 |
| 导入 `ImportSelector` 接口的实现类                 |
| 导入 `ImportBeanDefinitionRegistrar` 接口的实现类  |



| **说明**                                                                                                 |
| :------------------------------------------------------------------------------------------------------- |
| 所谓的导入其实就是将对象封装为 BeanDefinition，之后注册到 Spring 容器中交给 Spring 管理。即注册为 Bean！ |

# `@Import` 的三种使用方式

三种使用方式？在前面的 Spring 官网释义不是说四种吗？

其实在 Spring 底层源码中 `@Import` 注解解析方式只有三种！也就是导入 `ImportSelector` 接口的实现类、导入 `ImportBeanDefinitionRegistrar` 接口的实现类以及导入其他类。

## 导入普通类

这个很容易理解，就是导入一个普通的 Java 类：


```java
// Java 类
@Setter
@Getter
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -6411668546137854809L;

    private String name;

    private Integer age;
}

// Java 配置类
@Configuration
@Import(User.class)  // <===== 导入 User.class
public class Config {
}

// 启动类
public class Application {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
        User user = context.getBean(User.class);
        System.out.println(user.toString());
    }
}
```


运行示例后能够得到 `User.class` 的信息，说明 `User.class` 被 Spring 容器管理，即注册为 Bean。

`@Import` 注解除了能够导入普通的 Java 类还能导入一个基于 `@Configuration` 注解的配置类，下面开始说明。与之前的示例相同，只是增加了一个配置类 `DefinedConfig` ：


```java
// Java 类
@Setter
@Getter
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -6411668546137854809L;

    private String name;

    private Integer age;
}

// 定义配置类
@Configuration
public class DefinedConfig {

    // 注册 Bean
    @Bean
    public User user(){
        return new User();
    }
}

// Java 配置类
@Configuration
@Import(DefinedConfig.class)  // <===== 导入配置类
public class Config {
}

// 启动类
public class Application {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
        User user = context.getBean(User.class);
        System.out.println(user.toString());
    }
}
```


最后，能够得到 `User.class` 的 `hashCode` 。
## 导入 `ImportSelector` 的实现类
对于 ImportSelector 的原理这里不做具体说明，直接看下应用吧，定义一个 `ImportSelector`  接口的实现类： `UserImportSelector` ，演示需要我们同时实现类 `BeanFactoryAware` 接口。下面来看下具体示例：


```java
// Java 类
@Setter
@Getter
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -6411668546137854809L;

    private String name;

    private Integer age;
}

// ImportSelector 实现类
public class UserImportSelector implements ImportSelector, BeanFactoryAware {

    private BeanFactory beanFactory;

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        System.out.println("----------AnnotationMetadata-----------");
        importingClassMetadata.getAnnotationTypes().forEach(System.out::println);
        System.out.println("----------beanFactory-----------");
        System.out.println(beanFactory);

        try {
            beanFactory.getBean(User.class);
        } catch (Exception e){
            System.out.println("User.class 还没有被 Spring 管理");
        }

        return new String[]{User.class.getName()};
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}

// Java 配置类
@Configuration
@Import(UserImportSelector.class)  // <===== 导入 ImportSelector 实现类
public class Config {
}

// 启动类
public class Application {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
        User user = context.getBean(User.class);
        System.out.println(user.toString());
    }
}
```


输出结果如下：
```
----------AnnotationMetadata-----------
org.springframework.context.annotation.Configuration
org.springframework.context.annotation.Import
----------beanFactory-----------
org.springframework.beans.factory.support.DefaultListableBeanFactory@56ac3a89: defining beans [org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,org.springframework.context.event.internalEventListenerProcessor,org.springframework.context.event.internalEventListenerFactory,config]; root of factory hierarchy
User.class 还没有被 Spring 管理
User(name=null, age=null)
```
在 `UserImportSelector`  类中，特定使用了 `try cache` 语句块，在语句块中尝试获取 `User.class` 类型的 Bean，接口可想而知。之后，在 Spring 容器初始化完成后我们再次尝试获取该类型的 Bean： `User user = context.getBean(User.class);` 。这次成功获取，说明我们可以利用 `ImportSelector` 的实现类结合 `@Import` 来导入一个 Bean。


看到这个示例不知道你有没有感受到其中的魔力？如果你还没有任何体会这里可以给个小提示：**动态加载类（即注册Bean）。**
## 导入 ImportBeanDefinitionRegistrar 的实现类
从接口 `ImportBeanDefinitionRegistrar` 的名字不知道你没有看出该接口的具体作用，顾名思义：**导入一个Bean注册器 。**


我们都知道 Java 是一个面向对象编程，我们所编写的源代码最终被编译成字节码动态加载到虚拟机之中。那么如果我们想要获取我们的类该怎么办？Java 又是如何描述它的？答案是理所当然的： **`java.lang.Class<T>` **。所有的类或者说对象最终在代码层面都可以使用被称之为 `Class` 的类描述它。


在 Spring 中所有的对象都被称之为 `bean` ，你可以理解为这是它的规范。很多初学者以及工作一些年限的童鞋可能还不理解为什么是 `Bean` ，总结一句话就是： **所有被 Spring 管理的对象（你可以理解为 Java 类）都称之为 Bean** 。那么，在 Spring 中又是如何描述 Bean 的？那就是 `BeanDefinition` ！所有的 Bean 的描述信息都是在其中定义，比如 Bean 是单例的还是原型的、是不是懒加载等等。


现在，我们就不难理解 `ImportBeanDefinitionRegistrar` 接口了。


我们在使用该接口时通常会使用 `BeanDefinition` 的即可实现类，诸如： `GenericBeanDefinition` 、 `RootBeanDefinition` 和 `ChildBeanDefinition` 等等。当然，在 Spring 中 `BeanDefinition` 接口还直接或间接的提供了许多子类，具体可以查阅源码。


言归正传，现在来看看 `@Import` 注解如何配合 `ImportBeanDefinitionRegistrar` 接口一起使用：


```java
// Java 类
@Setter
@Getter
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -6411668546137854809L;

    private String name;

    private Integer age;

    // Setter And Getter
}

// ImportBeanDefinitionRegistrar 实现类
public class UserImportBeanDefinitionRegistor implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        // 这里使用最常使用的 BeanDefinition 实现类 GenericBeanDefinition
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(User.class);
        // 设置为原型
        beanDefinition.setScope("prototype");

        registry.registerBeanDefinition("user", beanDefinition);
    }
}

// Java 配置类
@Configuration
@Import(UserImportBeanDefinitionRegistor.class)  // <===== 导入 ImportBeanDefinitionRegistrar 实现类
public class Config {
}

// 启动类
public class Application {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
        User user1 = context.getBean(User.class);
        User user2 = context.getBean(User.class);
        System.out.println(user1.hashCode() + " ----- " + user2.hashCode());

    }
}
```


在实现类 `UserImportBeanDefinitionRegistor` 中，我们定义 `User.class` 类为原型 Bean，在启动测试类打印输出如下，得到了两个不同的 Hash 值，说明 `User.class` 成功导入 Spring 容器。


```
1881129850 ----- 1095293768
```
# 关于源码

上面一箩筐内容都是介绍 `@Import` 注解如何使用，而没有介绍其原理。首先要说明的是，Spring 对于 `@Import` 注解解析的方法是：

`org.springframework.context.annotation.ConfigurationClassParser#processImports`。

Spring 对于 `@Import` 注解的处理源于对 `@Configuration` 注解以及基于 XML 配置类的处理而处理的。Spring 对于 `@Configuration` 注解配置类的处理是由 `BeanFactoryPostProcess` 以及 `BeanDefinitionRegistryPostProcessor` 的实现类进行处理的，方法是：

`org.springframework.context.annotation.ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry`。

这个方法是 Spring 解析 `@Configuration` 的核心方法，方法如下：

```java
// org.springframework.context.annotation.ConfigurationClassPostProcessor

@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
   int registryId = System.identityHashCode(registry);
   if (this.registriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
   }
   if (this.factoriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
   }
   this.registriesPostProcessed.add(registryId);

   // Note
   // 核心工作: 处理配置类 @Configuration
   processConfigBeanDefinitions(registry);
}
```

从字面上理解 `ConfigurationClassPostProcessor` 这个类：配置类处理委托类！这样，基本上就知道这个类具体干什么的了。

这个方法继续调用当前类的 `processConfigBeanDefinitions()` 方法，这个方法

```java
// org.springframework.context.annotation.ConfigurationClassPostProcessor

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {

    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();

   // 获取所有的 BeanDefinition 名称
   String[] candidateNames = registry.getBeanDefinitionNames();

   // 循环得到所有 @Configuration 的 BeanDefinition
   for (String beanName : candidateNames) {

      // configCandidates.add(...)
   }

   // Return immediately if no @Configuration classes were found
   if (configCandidates.isEmpty()) {
      return;
   }

   // Parse each @Configuration class
   ConfigurationClassParser parser = new ConfigurationClassParser(
         this.metadataReaderFactory, this.problemReporter, this.environment,
         this.resourceLoader, this.componentScanBeanNameGenerator, registry);

   do {
      // 处理 Configuration 配置类
      // 包括 @ComponentScan、@Import、@ImportResource、@Bean
      parser.parse(candidates);
      parser.validate();
   }
   while (!candidates.isEmpty());

   // do something
}
```

直到这一步，还没有开始对 `@Configuration` 配置类进行解析，但是这个类真正实现的功能是循环 Spring 容器中的所有 BeanDefinition 对象，最终将所有的 `@Configuration` 注解 Bean 放到集合中。之后在这个方法中新创建了一个对象：`ConfigurationClassParser` ，这个类才是真正对 `@Configuration` 注解解析的信息类，这里进行调用 `parse` 方法，将得到的配置类集合传入该方法，该方法定义在  `ConfigurationClassParser` 类内部：

```java
// org.springframework.context.annotation.ConfigurationClassParser

// configCandidates 参数是所有的 @Configuration 配置类集合
public void parse(Set<BeanDefinitionHolder> configCandidates) {

   // 循环处理 @Configuration BeanDefinition
   for (BeanDefinitionHolder holder : configCandidates) {

      BeanDefinition bd = holder.getBeanDefinition();
      try {

         // parse(..) 方法内部处理
         // 处理 ImportSelector

         if (bd instanceof AnnotatedBeanDefinition) {
            parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
         } else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
            parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
         } else {
            parse(bd.getBeanClassName(), holder.getBeanName());
         }
      } catch (BeanDefinitionStoreException ex) {
      } catch (Throwable ex) {}
   }

   // 处理 DeferredImportSelector
   this.deferredImportSelectorHandler.process();
}
```

从这个方法中也基本上看懂了 Spring 对 `@Configuration` 注解配置类的处理流程：循环遍历 Spring 配置类（如基于注解的 `@Configuration`），之后一个一个的进行解析配置类中的内容。这个内容就包括对 `@Import` 注解的解析。

现在继续看下该方法内部三个 `if` 判断调用的 `parse` 方法。这个 `parse` 内部是个空壳方法，之后继续向下调用，最终对调用 `processConfigurationClass` 方法，该方法的参数就是配置类（如基于注解的 `@Configuration`）。

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {

   // do something...

   // Recursively process the configuration class and its superclass hierarchy.
   SourceClass sourceClass = asSourceClass(configClass);
   do {
      // 开始处理配置类中的内容
      sourceClass = doProcessConfigurationClass(configClass, sourceClass);
   }
   while (sourceClass != null);

   this.configurationClasses.put(configClass, configClass);
}
```

在 `doProcessConfigurationClass(configClass, sourceClass)` 方法中才是真正开始解析配置类中的内容，包括解析 `@PropertySource` 注解、`@ComponentScan` 注解、`@Import` 注解、`@Bean` 注解 等等。方法如下：

```java
@Nullable
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
      throws IOException {

   // Process any @ComponentScan annotations

   // Process any @Import annotations
   // 处理 @Import 注解
   processImports(configClass, sourceClass, getImports(sourceClass), true);

   // Process any @ImportResource annotations

   // do something...
   return null;
}
```

最终，就到了对 `@Import` 解析的核心方法：

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                     Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

   if (importCandidates.isEmpty()) {
      return;
   }

   // 链式循环调用?
   // 如果当前 Import 类已经被加入到 Stack 中表示链式调用, 抛出异常
   if (checkForCircularImports && isChainedImportOnStack(configClass)) {
      this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
   } else {
      this.importStack.push(configClass);
      try {
         for (SourceClass candidate : importCandidates) {
            // 处理 ImportSelector 实现类
            if (candidate.isAssignable(ImportSelector.class)) {
               // Candidate class is an ImportSelector -> delegate to it to determine imports
               Class<?> candidateClass = candidate.loadClass();
               ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
               ParserStrategyUtils.invokeAwareMethods(selector, this.environment, this.resourceLoader, this.registry);

               // DeferredImportSelector 是 ImportSelector 的子类, 所以首先判断该对象是否为 DeferredImportSelector 的实现类
               if (selector instanceof DeferredImportSelector) {
                  // 加入到 DeferredImportSelectorHandler#deferredImportSelectors 集合中
                  this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
               } else {
                  // 调用 ImportSelector#selectImports 方法, 获取指定的类的全限定名数组
                  String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                  Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);

                  // 递归调用,再次判断解析的类内部是否还存在 @Import 注解
                  processImports(configClass, currentSourceClass, importSourceClasses, false);
               }
            } else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
               // 处理 ImportBeanDefinitionRegistrar 实现类

               // Candidate class is an ImportBeanDefinitionRegistrar ->
               // delegate to it to register additional bean definitions
               Class<?> candidateClass = candidate.loadClass();
               ImportBeanDefinitionRegistrar registrar =
                     BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
               ParserStrategyUtils.invokeAwareMethods(
                     registrar, this.environment, this.resourceLoader, this.registry);

               // 将 ImportBeanDefinitionRegistrar 对象添加到集合中, 之后处理
               configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
            } else {
               // 处理普通类, 所谓的普通类包括配置类
               // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
               // process it as an @Configuration class
               this.importStack.registerImport(
                     currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
               processConfigurationClass(candidate.asConfigClass(configClass));
            }
         }
      } catch (BeanDefinitionStoreException ex) {
         throw ex;
      } catch (Throwable ex) {
         throw new BeanDefinitionStoreException(
               "Failed to process import candidates for configuration class [" +
                     configClass.getMetadata().getClassName() + "]", ex);
      } finally {
         this.importStack.pop();
      }
   }
}
```

看到这里，基本上就知道了底层源码是 Spring 对于 `ImportSelector` 接口以及 `ImportBeanDefinitionRegistrar` 接口的解析，这里会专门在一篇中进行介绍，这里不做过多说明。

对于普通方法的处理，底层又再次递归调用 `processConfigurationClass(ConfigurationClass configClass)` 方法进行循环处理。

这里只是笔者自己的了解，在实际理解中建议跟着本文源码部分对着源码一步一步的进行调试，说不定会有自己的见解呢~

# 写在最后
以上就是 Spring `@Import` 注解的基本的使用方式，该注解的主要作用就是导入一个普通类或者配置为为 Bean。上面只是基本的说明示例，具体还要在业务层面去实践。


题外话，Spring 自 3.0 之后提供了基于 JavaConfig 的配置形式。在之前的版本使用的都是基于 XML 的配置形式，当然 Spring 在不断更新但是依然没有舍弃 XML。各有各的优势，通常我们都是两者结合使用，尤其是在 SpringBoot 中更常使用。因此，Spring 除了提供 `@Import` 注解之外还提供了另外一个注解： **`@ImportResource`** 。该注解与 `@Import` 不同，但是具有相同的功能： **`@Import` 导入的是 Java 类，而 `@ImportResource` 导入的是 XML。**在业务中，如果想要导入基于 XML 的配置，我们只需要在配置类上使用该注解即可。比如在 `resources` 目录下有一个 `spring-aop.xml` 的配置文件，在配置类中我们使用该注解导入即可：


```java
@Configuration
@ImportResource("classpath:spring-aop.xml") // <===== 导入 XML, 注意要加 classpath
public class Config {
}
```
