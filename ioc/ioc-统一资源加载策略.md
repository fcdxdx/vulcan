# ioc-统一资源加载策略

## 统一资源 ：Resource

`org.springframework.core.io.Resource` 为spring框架所有资源的抽象和访问接口，它继承`org.springframework.core.io.InputStreamSource`接口，作为所有资源的统一抽象，Resource定义了一些通用的方法，由子类`AbstractResource`提供统一的默认实现，定义如下

[AbstractResource源码](https://github.com/fcdxdx/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/AbstractResource.java)

类结构图

![image-20190426152127824](https://raw.githubusercontent.com/fcdxdx/tuchuang/master/image-20190426152127824.png)

上图可看出，Resource根据资源的不同提供不同的具体实现，如下

* FileSystemResource：对`java.io.File`类型的资源封装，只要是跟File打交道，基本上与FileSystemResource也可以打交道
* ByteArrayResource：对字节数组提供的数据封装，如通过InputStream形式访问该类型的资源，改实现会根据字节数组的数据构造一个相应的ByteArrayInputStream
* UrlResource：对`java.net.URL`类型资源的封装。内部委派URL进行具体的资源操作。
* ClassPathResource：class path类型资源的实现，使用给定的ClassLoader或者给定的Class来加载资源。
* InputStreamResource：将给定的InputStream作为资源的Resource的实现类。

`AbstractResource`是Resource接口的默认实现，它实现了Resource接口的大部分的公共实现，源码如下：

[AbstractResource源码](https://github.com/fcdxdx/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/io/AbstractResource.java)

**如果我们想要自定义Resource，记住不要实现Resource接口，应该继承AbstractResource抽象类，然后根据实际情况覆盖相应的方法**



## 统一资源定位：ResourceLoader

spring将资源的定义和资源的加载区分开，Resource定义了统一的资源，ResourceLoader定义了资源的统一加载。

`org.springframework.core.io.ResourceLoader` 为spring资源加载的统一抽象，具体的资源加载则由相应的实现类来完成，我们可以将ResourceLoader称作为统一资源定位器。

```java
public interface ResourceLoader {
    String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
    Resource getResource(String location);
    ClassLoader getClassLoader();
}
```

ResourceLoader接口提供两个方法：`getResource()`、`getClassLoader()`

`getResource()`根据所提供的资源路径location返回Resource实例，但是不确保Resource一定存在，需要调用`Resource.exist()`方法判断，该方法支持以下模式的资源加载：

* URL位置资源，如“file:C:/test.dat”
* ClasPath位置资源，如“classpath:test.dat”
* 相对路径资源，如“WEB-INF/test.dat”，此时返回的Resource实例根据实现不同而不同

该方法的主要实现是在其子类`DefaultResourceLoader`中实现。

`getClassLoader`返回ClassLoader实例，对于想要获取ResourceLoader使用的ClassLoader用户来说，可以直接调用该方法来获取，在分析Resource时，提到了一个类ClassPathResource，这个类是可以根据指定的ClassLoader来加载资源的。

作为Spring统一的资源加载器，它提供了统一的抽象，具体的实现则由相应的子类来实现，具体结构如下：

![jiegoutu](https://raw.githubusercontent.com/fcdxdx/tuchuang/master/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190428143100.png)

### DefaultResourceLoader

与DefaultResource类似，DefaultResourceLoader是ResourceLoader的默认实现，它接受ClassLoader作为构造函数的参数或者使用不带参数的构造函数，在使用不带参数的构造函数时，使用ClassLoader为默认的ClassLoader（一般为`Thread/currentThread().getContextClassLoader()`），可以通过`ClassUnits.getDefaultClassLoader()`获取。当然也可以调用`setClassLoader()`方法进行后续设置。

```java
public DefaultResourceLoader() {
     this.classLoader = ClassUtils.getDefaultClassLoader();
 }
public DefaultResourceLoader(@Nullable ClassLoader classLoader) {
     this.classLoader = classLoader;
 }
public void setClassLoader(@Nullable ClassLoader classLoader) {
     this.classLoader = classLoader;
 }
 @Override
 @Nullable
public ClassLoader getClassLoader() {
   return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
    }
```

