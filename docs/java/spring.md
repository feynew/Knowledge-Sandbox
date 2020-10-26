# 概览
## 1. Spring框架：简化java开发
- DI——松耦合
- AOP——把遍布于应用各处的功能分离出来成为可重用的组件
- 消除样板式代码
## 2.bean的生命周期
在传统的Java应用中，bean的生命周期很简单，使用Java关键字 new 进行Bean 的实例化，然后该Bean 就能够使用了。一旦bean不再被使用，则由Java自动进行垃圾回收。

相比之下，Spring管理Bean的生命周期就复杂多了，正确理解Bean 的生命周期非常重要，因为Spring对Bean的管理可扩展性非常强，下图展示了一个Bean装载到spring应用上下文中的一个典型的生命周期过程。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13394a841c57450f8729c974a30e9280~tplv-k3u1fbpfcp-watermark.image)
如上图所示，Bean 的生命周期还是比较复杂的，下面来对上图每一个步骤做文字描述:

1. Spring启动，查找并加载需要被Spring管理的bean，进行Bean的实例化
2. Bean实例化后对将Bean的引入和值注入到Bean的属性中
3. 如果Bean实现了BeanNameAware接口，Spring将Bean的Id传递给setBeanName()方法
4. 如果Bean实现了BeanFactoryAware接口，Spring将调用setBeanFactory()方法，将BeanFactory容器实例传入
5. 如果Bean实现了ApplicationContextAware接口，Spring将调用Bean的setApplicationContext()方法，将bean所在应用上下文引用传入进来
6. 如果Bean实现了BeanPostProcessor接口，Spring就将调用他们的postProcessBeforeInitialization()方法
7. 如果Bean 实现了InitializingBean接口，Spring将调用他们的afterPropertiesSet()方法。类似的，如果bean使用init-method声明了初始化方法，该方法也会被调用
8. 如果Bean 实现了BeanPostProcessor接口，Spring就将调用他们的postProcessAfterInitialization()方法
9. 此时，Bean已经准备就绪，可以被应用程序使用了。他们将一直驻留在应用上下文中，直到应用上下文被销毁。
10. 如果bean实现了DisposableBean接口，Spring将调用它的destory()接口方法，同样，如果bean使用了destory-method 声明销毁方法，该方法也会被调用。
## 3.Spring框架提供的模块
Spring框架由6个定义良好的模块分类组成

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae93f1cad1f846d7817bc2971578e561~tplv-k3u1fbpfcp-watermark.image)

## 4.bean的作用域
默认情况下，Spring应用上下文中所有的bean都是作为单例的形式创建的。多数情况下，单例bean是很理想的方案，这些对象是无状态的且可以在应用程序中反复使用。有时候，对象会保持一些状态，因此重用是不安全的。

#### 4.1 bean的作用域
Spring定义了多种作用域， 包括
- 单例（Singleton）：在整个应用中，只创建bean的一个实例。
- 原型（Prototype）：每次注入或者通过Spring应用上下文获取时，都会创建一个新的bean实例。
- 会话（Session）：在web应用中，为每个会话创建一个bean实例。
- 请求（Request）：在web应用中，为每个请求创建一个bean实例。

#### 4.2 bean作用域的指定方法
（1）注解方式：在bean的类上使用@Scope注解
```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class TestBean { ... }
```
（2）xml方式
```xml
<bean id="testBean" 
      class="com.myapp.TestBean" 
      scope="prototype" />
```
#### 4.3 使用会话和请求作用域
当bean的作用域是会话时，spring会为每个会话创建一个实例。当会话或请求作用域的bean注入到单例bean中时，可能会有问题。下面以购物车为例来说明这个场景。假设我们要将ShoppingCart bean注入到单例StoreService bean中：
```java
@Component
public class StoreService { 
    @Autowired
    private ShoppingCart shoppingCart;
    ...
}
```
因为单例bean在spring应用上下文加载时创建，当它创建时，spring会试图将shoppingCart bean注入到storeService中，但是shoppingCart是会话作用域，此时并不存在。直到某个用户进入系统，创建了会话之后，才会出现ShoppingCart实例。另外，系统中将会有多个ShoppingCart实例，我们并不想让spring注入某个固定的ShoppingCart实例到StoreService中。我们希望的是当StoreService处理购物车功能是，它所使用的ShoppingCart实例恰好是当前会话对应的那一个。
Spring并不会将实际的ShoppingCart bean注入到StoreService中，而是注入一个ShoppingCart的代理，当StoreService调用ShoppingCart的方法时，代理会对其进行懒解析并将调用委托给会话作用域内真正的ShoppingCart bean。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a851bb9a64de457c8a6b220c6788277c~tplv-k3u1fbpfcp-watermark.image)

@Scope提供了proxyMode属性来解决这个问题。proxyMode指定了是基于接口代理还是类代理。
- ScopedProxyMode.INTERFACES：如果ShoppingCart是个接口，则设置成接口代理，此时代理可以直接实现该接口。
- ScopedProxyMode.TARGET_CLASS：如果ShoppingCart是个具体的类，spring就没有办法创建基于接口的代理了，此时，它必须使用CGLib来生成基于类的代理。
声明作用域代理的方法有两种：

（1）注解方式
```java
@Component
@Scope(value=ConfigurableBeanFactory.SCOPE_PROTOTYPE,
       proxyMode=ScopedProxyMode.INTERFACES)
public class TestBean { ... }
```
（2）xml方式
```xml
<bean id="testBean" 
      class="com.myapp.TestBean" 
      scope="prototype" >
   <aop:scoped-proxy />
</bean>
```
<aop:scoped-proxy />会告诉spring为bean创建一个作用域代理。默认情况下，它会使用CGLib创建目标类的代理，如果想生成基于接口的代理，可以将proxy-target-class设置成false:
```xml
<bean id="testBean" 
      class="com.myapp.TestBean" 
      scope="prototype" >
   <aop:scoped-proxy proxy-target-class=false/>
</bean>
```
注意：为了使用<aop/>元素，我们必须在xml配置中声明spring的aop命名空间。

> 部分内容来源于https://www.cnblogs.com/javazhiyin/p/10905294.html
