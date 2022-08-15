### 对于SpringBoot的理解：

SpringBoot 的特性，是简单、快捷的能够创建一个spring应用。

使用者不用关注繁杂的配置文件，这个工作由SpringBoot代替我们解决。



关键在于

**SpringBoot是如何处理的：**

1、标注 **@SpringBootApplication** 注解的主类（SpringBoot启动类）；

2、@SpringBootApplication 是个复合注解，其中包含

​		**第一个注解：@EnbleAutoConfigurtion ：帮助我们开启自动配置功能。**

​		其原理依靠

​		①、**@AutoConfigurationPackage**之中的**@Import**(AutoConfigurationPackages.Registrar.class)

帮助SpirngBoot把主程序类包下的所有组件用Registrar方法注册进容器中。

(源代码如下)

````java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
   Registrar() {
   }
   
   //Registrar方法传递两个参数，第一个是注解的源信息AnnotationMetadata，这个注解标在主程序类MainApplication上（合成注解层层传递）
   public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
       //Registrar方法利用注解源信息获取到主程序MainApplication所在包名com.dlhjw.boot，封装成数组，注册进容器里。
       AutoConfigurationPackages.register(registry, (String[])(new AutoConfigurationPackages.PackageImports(metadata)).getPackageNames().toArray(new String[0]));
   }

   public Set<Object> determineImports(AnnotationMetadata metadata) {
       return Collections.singleton(new AutoConfigurationPackages.PackageImports(metadata));
   }
}
````

​		**②、@Import(EnableAutoConfigurationImportSelector.class)【重要】：引入自动配置类**

​			在SpringBoot初始启动时导入127个自动配置类，按照条件配装规则@conditional,按需配置。



(源代码分析)  

**总体上：利用Selector机制给容器批量导入自动配置类；（底层 -> 实现）**

1. 首先从META-INF/spring.factories位置加载一个文件。即默认扫描当前系统里面所有META-INF/spring.factories位置的文件。其中最重要的是spring-boot-autoconfigure-2.3.4.RELEASE.jar包里的META-INF/spring.factories（SpringBoot兼容全场景的127个自动配置类就在这里，即xxxxAutoConfiguration）；

2. 接着使用Spring的工厂加载类loadSpringFactories得到所有的组件；

3. 然后调用getCandidateConfigurations()获取到所有需要导入到容器中的配置类（默认导入导容器中的127个全类名组件）

4. 接着利用getAutoConfigurationEntry(annotationMetadata)方法获取自动配置集合

5. 【核心】最后对getAutoConfigurationEntry(annotationMetadata)获取到的配置进行封装，封装成selectImports(AnnotationMetadata am)方法，返回String数组，数组里说明了需要导入的自动配置类（组件）。

   

​		**第二个注解：@SpringBootConfiguration**

- 配置注解；

- 标注类上；

- 底层是一个`@Configuration`，代表当前类是一个配置类。在这里指核心配置类。

  

​		**第三个注解：@ComponentScan**

- 开启包扫描；
- 标注类上；
- 可以指定扫描路径，扫描到的包里的注解才能生效。在这里自定义了两个扫描器。
  - 例：`@ComponentScan(basePackages = {"com.dlhjw"})`