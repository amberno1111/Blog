# 一、背景

## 1.1 Spring中的@Value注解
在Spring中，一般在`application.properties` 文件中指定配置项的值
```properties
spring.demo.name=spring-demo-app-name
```

然后使用`@Value` +占位符的方式来注入配置项，Spring就能自动把这个值注入进来：
```java
@Configuration  
public class DemoConfiguration {  
  
    @Value("${spring.demo.name}")  
    private String name;
    
	@Bean  
	public String demoName() {
		// spring启动了以后，这里输出的就是 spring-demo-app-name
	    System.out.println(name);  
	    return name;  
	}
}
```


## 1.2 @Value无法注入
Spring的这个`@Value`注解的使用是比较简单的，一般很少使用出错。但是我们却发现在某次代码重构以后，标记了`@Value`注解的字段无法注入配置项的值了，而且并不是所有的配置项都无法注入，而是只有某一个类里的字段值无法用`@Value`注入。这个类的代码如下：
```java
@Configuration  
public class MybatisConfiguration {  
  
    @Value("${mapper.package}")  
    private String basePackage;
  
    @Bean  
    public MapperScannerConfigure mapperScannerConfigure() {  
        MapperScannerConfigure mapperScannerConfigure = new MapperScannerConfigure();
        // 运行起来的时候，这个this.basePackage得到的值是null 
        mapperScannerConfigure.setBasePackage(this.basePackage);
        // xxxxx 这里省略一些无关的代码
        return mapperScannerConfigure; 
    }
}
```

在`application.properties`文件中也有对应的配置：
```properties
mapper.package=com.xxxx.xxx.xxx
```

使用方式肯定是没错的，毕竟项目里其他地方使用`@Value`也正常注入了配置项的值。那就没办法了，只能研究下Spring的源码了，看看Spring是怎么把配置项注入进来的。


# 二、`@Value`配置项注入的原理

先debug了下能正常注入配置项字段的Bean，发现通过`@Value`标记的字段，是在Spring的`org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessProperties` 这个方法中处理的：
```java
@Override  
public PropertyValues postProcessProperties(PropertyValues pvs, 
	Object bean, String beanName) {
	// Step1. 第一步找到这个bean的metadata
   InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);  
   try {  
	   // Step2. 第二步直接执行注入，会把配置项的值通过反射的方式设置为字段的值
      metadata.inject(bean, beanName, pvs);  
   }   catch (BeanCreationException ex) {  
      throw ex;  
   }   catch (Throwable ex) {  
      throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);  
   }   
   return pvs;  
}
```

## 2.1 `findAutowiringMetadata`找到被`@Value`标记的字段
先来看一下这个`findAutowiringMetadata`方法，从名字上来看跟`@Value`毫无关系，他的作用其实是去扫描这个Bean当中所有被`@Autowired`和`@Value`标记的字段。
```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {  
   // Fall back to class name as cache key, for backwards compatibility with custom callers.  
   String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());  
   // Quick check on the concurrent map first, with minimal locking.
   // 这里是个缓存，不需要关心，直接看后面构造  InjectionMetadata 的地方
   InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);  
   if (InjectionMetadata.needsRefresh(metadata, clazz)) {  
	   synchronized (this.injectionMetadataCache) {  
		   metadata = this.injectionMetadataCache.get(cacheKey);  
	         if (InjectionMetadata.needsRefresh(metadata, clazz)) {  
             if (metadata != null) {  
               metadata.clear(pvs);  
             }
	         // 这个方法里构造出 InjectionMetadata           
	         metadata = buildAutowiringMetadata(clazz);  
	         this.injectionMetadataCache.put(cacheKey, metadata);  
			}     
		}   
	}   
    return metadata;  
}

private InjectionMetadata buildAutowiringMetadata(Class<?> clazz) {  
   do {  
      final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();  
      // ... 中间省略一些无关的代码
      ReflectionUtils.doWithLocalFields(targetClass, field -> {
		//可以看到这里是通过反射来拿所有的字段，而这个字段的过滤条件则是这个
		// findAutowiredAnnotation方法  
         MergedAnnotation<?> ann = findAutowiredAnnotation(field);
    } 
    // ... 中间省略一些无关的代码
   }   while (targetClass != null && targetClass != Object.class);  
   return InjectionMetadata.forElements(elements, clazz);  
}
@Nullable  
private MergedAnnotation<?> findAutowiredAnnotation(AccessibleObject ao) {  
   MergedAnnotations annotations = MergedAnnotations.from(ao);
   // 可以看到，最主要是通过这个 this.autowiredAnnotationTypes 来进行过滤的
   // 实际上这是个List，里面的元素有两个，分别是Autowired.class 和 Value.class
   // 所以这个方法能找到所有用@Autowired和@Value标记的字段
   for (Class<? extends Annotation> type : this.autowiredAnnotationTypes) {  
      MergedAnnotation<?> annotation = annotations.get(type);  
      if (annotation.isPresent()) {  
         return annotation;  
      }   }   return null;  
}
// 而 this.autowiredAnnotationTypes这个列表里的元素是在构造函数里加进去的
@SuppressWarnings("unchecked")  
public AutowiredAnnotationBeanPostProcessor() {
   // 看到这里谜题就解开了，AutowiredAnnotationBeanPostProcessor 确实能找到@Value标记的字段
   this.autowiredAnnotationTypes.add(Autowired.class);  
   this.autowiredAnnotationTypes.add(Value.class);
   // ... 中间省略一些无关的代码
}										
```

## 2.2 通过反射给字段赋值

看这个方法：`org.springframework.beans.factory.annotation.InjectionMetadata#inject`
```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {  
   Collection<InjectedElement> checkedElements = this.checkedElements;  
   Collection<InjectedElement> elementsToIterate =  
         (checkedElements != null ? checkedElements : this.injectedElements);  
   if (!elementsToIterate.isEmpty()) {  
      for (InjectedElement element : elementsToIterate) {
	     // 这里会遍历每个InjectedElement做处理，然后我们InjectedElement有两个实现类
	     // 分别是AutowiredFieldElement和AutowiredMethodElement
	     // 我们这里是注入字段，所以就直接看AutowiredFieldElement的inject方法即可
         element.inject(target, beanName, pvs);  
      }   
    }
}
// org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject 
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {  
   Field field = (Field) this.member;  
   Object value;  
   if (this.cached) {  
      try {  
         value = resolvedCachedArgument(beanName, this.cachedFieldValue);  
      } catch (NoSuchBeanDefinitionException ex) {  
         // Unexpected removal of target bean for cached argument -> re-resolve  
         value = resolveFieldValue(field, bean, beanName);  
      }
    } else {  
      value = resolveFieldValue(field, bean, beanName);  
   }
   if (value != null) {
	  // 这里直接通过反射赋值，就完成了配置项注入到被@Value标记的字段上的任务  
      ReflectionUtils.makeAccessible(field);  
      field.set(bean, value);  
   }
}
``` 


# 三、提前初始化的Bean

了解了原理之后，我们开始研究为什么`MybatisConfiguration`里用`@Value`标记的字段没有被注入配置项的值，所以我们在`org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessProperties` 这个方法上打了条件断点，但是发现并没有走进来。也就是说，这个Bean的处理并没有经过`AutowiredAnnotationBeanPostProcessor`处理，这就奇怪了。

那么就干脆把断点放在最前面，从`org.springframework.context.support.AbstractApplicationContext#refresh`开始一步步debug下来，发现是通过走到`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean`方法里，再去调用`AutowiredAnnotationBeanPostProcessor`的。从堆栈上来看，能正常注入`@Value`字段值的Bean和不能正常注入`@Value`字段值的Bean，进入这个populateBean方法的时机是不一样的：
```java
public void refresh() throws BeansException, IllegalStateException {  
   synchronized (this.startupShutdownMonitor) {  
      StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh"); 
      // .....
      try {  
         // .....
         // MybaticConfiguration 从这个地方进入到populateBean方法
         // 此时还没有注册AutowiredAnnotationBeanPostProcessor这个processor
         // 那么pupulateBean方法中自然无法调用AutowiredAnnotationBeanPostProcessor来执行
         invokeBeanFactoryPostProcessors(beanFactory);  
  
         // 这个地方才会把AutowiredAnnotationBeanPostProcessor注册进去
         registerBeanPostProcessors(beanFactory);
         // .....
         
         // 能正常注入@Value字段值的Bean从这个地方为入口进入到populateBean方法
         // 此时pupulateBean方法中可以调用到AutowiredAnnotationBeanPostProcessor来执行
         finishBeanFactoryInitialization(beanFactory);  
         // .....
      }  catch (BeansException ex) {}
}
```

看到这里，自然的就会产生两个疑问：
- 为什么`MybaticConfiguration`执行`populateBean`方法更早？
- `MybaticConfiguration`在`finishBeanFactoryInitialization`方法里不会再次执行`populateBean`方法么？再执行一遍就能调用到`AutowiredAnnotationBeanPostProcessor`然后完成字段值的注入了么？

先回答第二个问题：继续翻看源码就会发现`populateBean`是通过`org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean`方法为入口执行进来的，这是个创建bean的方法，肯定不可能执行两遍的。

也就是说，能正常注入`@Value`字段值的Bean是在执行`finishBeanFactoryInitialization`方法才创建出来的。这时候第一个问题其实就变成了，`MybaticConfiguration`这个Bean为什么会提前被创建？？

再继续从`invokeBeanFactoryPostProcessors`方法往下翻，就会有结论：
```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   // 从这个方法进去
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());  
  
   //.....
   }
}
public static void invokeBeanFactoryPostProcessors(  
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {  
      // .....
      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.  
      boolean reiterate = true;  
      while (reiterate) {  
         reiterate = false;
         // 在这里，去拿到所有的BeanDefinitionRegistryPostProcessor.class bean
         // 结果发现，拿到了我们在MybaticConfiguration里定义的那个bean MapperScannerConfigure
         // 翻看了一下这个类的定义，确实是实现了BeanDefinitionRegistryPostProcessor
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);  
         for (String ppName : postProcessorNames) {  
            if (!processedBeans.contains(ppName)) {
               // 然后开始创建实现了BeanDefinitionRegistryPostProcessor的这些Bean
               // 自然也包括MapperScannerConfigure，由于这个Bean是被定义在
               // MybaticConfiguration里的
               // 所以Spring就先创建了MybaticConfiguration这个Bean
               // 这就解释了MybaticConfiguration被提前创建的原因
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            }         
        }         
      }  

}
```


**看到这里，这个问题的原因就很明显了，因为`MapperScannerConfigure`实现了`BeanDefinitionRegistryPostProcessor`，它需要被提前创建，所以定义这个Bean的那个`Configuration`Bean（`MybaticConfiguration`）也需要被提前创建。而这个提前创建的时机是比较早的，此时`AutowiredAnnotationBeanPostProcessor`并没有被注册进来，所以执行不到，也就导致`MybaticConfiguration`里标记了`@Value`的字段无法被注入配置项的值。**


# 四、解决方案
既然发现了问题，要怎么解决呢？？其实方案很简单，回头去看`org.springframework.context.support.AbstractApplicationContext#refresh`方法，可以发现其实在`prepareRefresh`方法里，已经把`Environment`创建好了。对于SpringBoot应用来说，这个`Environment`其实创建和初始化的时机更早。

也就是说，在创建`MybatisConfiguration`这个Bean之前，`Environment`已经处理好了，那么可以手动调用`Environment`来读配置项的值，代码修改如下：
```java
@Configuration  
public class MybatisConfiguration implements EnvironmentAware {

	private Environment environment;

	@Override  
	public void setEnvironment(Environment environment) {  
	    this.environment = environment;  
	}
  
    @Bean  
    public MapperScannerConfigure mapperScannerConfigure() {  
        MapperScannerConfigure mapperScannerConfigure = new MapperScannerConfigure();
        // 从environment里可以直接取到这个配置项的值
        String basePackage = this.environment.resolvePlaceholders("${mapper.package}");
        mapperScannerConfigure.setBasePackage(basePackage);
        // xxxxx 这里省略一些无关的代码
        return mapperScannerConfigure; 
    }
}
```




