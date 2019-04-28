转载：[Java 技术驿站](http://cmsblogs.com/) » [【死磕 Spring】—– IOC 之 Spring 统一资源加载策略](http://cmsblogs.com/?p=2656)

---



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

与DefaultResource类似，DefaultResourceLoader是ResourceLoader的默认实现，它接受ClassLoader作为构造函数的参数或者使用不带参数的构造函数，在使用不带参数的构造函数时，使用ClassLoader为默认的ClassLoader（一般为`Thread/currentThread().getContextClassLoader()`），可以通过`ClassUnits.getDefaultClassLoader()`获取。当然也可以调用`setClassLoader()`方法进行后续设置。如下：

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

ResourceLoader中最核心的方法为`getResource()`，它根据提供的location返回相应的Resource，而DefaultResourceLoader对该方法提供了核心实现（它的两个子类都没有提供覆盖该方法，所以可以断定ResourceLoader的资源加载策略就封装DefaultResourceLoader中），如下：



```java
/**
	 * @Author jojo.wang
	 * @Description ResourceLoader 中最核心的方法为
	 * 它根据提供的 location 返回相应的 Resource
	 * DefaultResourceLoader 对该方法提供了核心实现
	 * (它的两个子类都没有提供覆盖该方法，所以可以断定ResourceLoader 的资源加载策略就封装 DefaultResourceLoader中)
	 * @Date 13:21 2019-04-26
	 * @param location
	 * @return org.springframework.core.io.Resource
	 **/
	@Override
	public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");
		for (ProtocolResolver protocolResolver : this.protocolResolvers) {
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}
		if (location.startsWith("/")) {
			return getResourceByPath(location);
		}
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		else {
			try {
				// Try to parse the location as a URL...
				URL url = new URL(location);
				return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
			}
			catch (MalformedURLException ex) {
				// No URL -> resolve as resource path.
				return getResourceByPath(location);
			}
		}
	}
```

```java
protected Resource getResourceByPath(String path) {
		return new ClassPathContextResource(path, getClassLoader());
	}
```

```java
public ClassPathResource(String path, @Nullable ClassLoader classLoader) {
		Assert.notNull(path, "Path must not be null");
		String pathToUse = StringUtils.cleanPath(path);
		if (pathToUse.startsWith("/")) {
			pathToUse = pathToUse.substring(1);
		}
		this.path = pathToUse;
		this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
	}
```

```java
protected Resource getResourceByPath(String path) {
	return new ClassPathContextResource(path, getClassLoader());
}
```



首先通过ProtocolResolver来加载资源，成功返回Resource，否则调用一下逻辑：

* 若location以 / 开头，则调用`getResourceByPath()`构造ClassPathContextResource类型资源并返回
* 若location以classpath:开头，则构造ClassPathResource类型资源并返回，在构造该资源时，通过`getClassLoader()`获取当前ClassLoader
* 构造URL，尝试通过它进行资源定位，若没有抛出`MalformedURLException `异常，则判断是否未FileUrl，如果是则构造FileUrlResource类型资源，否则构造UrlResource类型资源。若在加载过程中抛出`MalformedURLException `异常，则委派`getResourceByPath()`实现资源定位加载

ProtocolResolver，用户自定义协议资源解决策略，作为DefaultResourceLoader的SPI，它允许用户自定义资源加载协议，而不需要继承ResourceLoader子类。在介绍Resource时，提到如果要实现自定义的Resource，我们只需要继承AbstractResource即可，但有了ProtocolResolver后，我们不需要直接继承DefaultResourceLoader，改为实现ProtocolResolver接口也可以实现自定义的ResourceLoader。

ProtocolResolver接口，仅有一个方法`Resource resolve(String location, ResourceLoader resourceLoader);`,该方法接收两个参数：资源路径location，指定的加载器ResourceLoader，返回为相应的Resource。在Spring中你会发现该接口并没有实现类，它需要用户自定义，自定义的Resolver如何加载Spring体系呢？调用`DefaultResourceLoader.addProtocolResolver()`即可，如下：

```java
/**
 * Register the given resolver with this resource loader, allowing for
 * additional protocols to be handled.
 * <p>Any such resolver will be invoked ahead of this loader's standard
 * @since 4.3
 * @see #getProtocolResolvers()
 */
public void addProtocolResolver(ProtocolResolver resolver) {
	Assert.notNull(resolver, "ProtocolResolver must not be null");
	this.protocolResolvers.add(resolver);
}
```

下面是演示DefaultResourceLoader加载资源的具体策略，代码如下：

```java
public class DefaultResourceLoaderITest {
    public static void main(String[] args) {
        ResourceLoader resourceLoader = new DefaultResourceLoader();
        Resource file1 = resourceLoader.getResource("D:/spring/vulcan/summary.md");
        System.out.println("file1 is FileSystemResource:"+(file1 instanceof FileSystemResource));

        Resource file2 = resourceLoader.getResource("/spring/vulcan/summary.md");
        System.out.println("file2 is ClassPathResource:"+(file2 instanceof ClassPathResource));

        Resource url = resourceLoader.getResource("file:/spring/vulcan/summary.md");
        System.out.println("url is UrlResource:"+(url instanceof UrlResource));

        Resource url2 = resourceLoader.getResource("http://www.baidu.com");
        System.out.println("url2 is UrlResource:"+(url2 instanceof UrlResource));
    }
}
```

运行结果

```bash
file1 is FileSystemResource:false
file2 is ClassPathResource:true
url is UrlResource:true
url2 is UrlResource:true
```

对于file1我们更加希望的是FileSystemResource资源类型，但是事与愿违，他是ClassPathResource类型。在`getResource()`资源加载策略中，我们知道`D:/spring/vulcan/summary.md`资源其实在该方法中没有响应的资源类型，那么他就会在抛出MalformedURLException 异常是通过`getResourceByPath()`构造一个ClassPathResource类型的资源。而制定有协议前缀的资源路径，则通过URL就可以定义，返回的都是UrlResource类型。

### FileSystemResourceLoader

从上面的实例我们看到，其实DefaultResourceLoader对`getResourceByPath(String)`方法处理其实不是很恰当，这个时候我们可以使用FileSystemResourceLoader，它继承DefaultResourceLoader且覆盖了`getResourceByPath(String)`方法，使之从文件系统加载资源并以FileSystemResource类型返回，这样我们就可以得到想要的资源类型了，如下：

```java
 @Override
    protected Resource getResourceByPath(String path) {
        if (path.startsWith("/")) {
            path = path.substring(1);
        }
        return new FileSystemContextResource(path);
    }
```

FileSystemContextResource 为 FileSystemResourceLoader 的内部类，它继承 FileSystemResource。

```java
private static class FileSystemContextResource extends FileSystemResource implements ContextResource {
        public FileSystemContextResource(String path) {
            super(path);
        }
        @Override
        public String getPathWithinContext() {
            return getPath();
        }
    }
```

在构造器中也是调用 FileSystemResource 的构造方法来构造 FileSystemContextResource 的。

如果将上面的示例将 DefaultResourceLoader 改为 FileSystemContextResource ，则 fileResource1 则为 FileSystemResource。



### ResourcePatternResolver

ResourceLoader的`Resource getResource(String location)` 每次只能根据location返回一个Resource，当需要加载多个资源时，我们除了多次调用`getResource()`外别无它法。ResourcePatternResolver 是ResourceLoader的扩展，它支持根据指定的资源路径匹配模式每次返回多个Resource实例，其定义如下：

```java
public interface ResourcePatternResolver extends ResourceLoader {
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";
    Resource[] getResources(String locationPattern) throws IOException;
}
```

ResourcePatternResolver 在ResourceLoader的基础上增加了`getResources(String locationPattern)`，以支持根据路径匹配模式返回多个Resource实例，同时也新增了一中新的协议前缀`classpath*:`，该协议前缀由子类负责实现。

PathMatchingResourcePatternResolver 为 ResourcePatternResolver 最常用的子类，它除了支持 ResourceLoader 和 ResourcePatternResolver 新增的 `classpath*:` 前缀外，还支持 Ant 风格的路径匹配模式（类似于 `**/*.xml`）。

PathMatchingResourcePatternResolver 提供了三个构造方法，如下：

```java
public PathMatchingResourcePatternResolver() {
        this.resourceLoader = new DefaultResourceLoader();
    }
    public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
        Assert.notNull(resourceLoader, "ResourceLoader must not be null");
        this.resourceLoader = resourceLoader;
    }
    public PathMatchingResourcePatternResolver(@Nullable ClassLoader classLoader) {
        this.resourceLoader = new DefaultResourceLoader(classLoader);
    }
```

PathMatchingResourcePatternResolver 在实例化的时候，可以指定一个 ResourceLoader，如果不指定的话，它会在内部构造一个 DefaultResourceLoader。

#### Resource getResource(String location)

```java
@Override
    public Resource getResource(String location) {
        return getResourceLoader().getResource(location);
    }
```

`getResource()` 方法直接委托给相应的 ResourceLoader 来实现，所以如果我们在实例化的 PathMatchingResourcePatternResolver 的时候，如果不知道 ResourceLoader ，那么在加载资源时，其实就是 DefaultResourceLoader 的过程。其实在下面介绍的 `Resource[] getResources(String locationPattern)` 也相同，只不过返回的资源时多个而已。

#### Resource[] getResources(String locationPattern)

```java
@Override
	public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");
		//以classpath*:开头
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
			// a class path resource (multiple resources for same name possible)
			//路径包含通配符
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
				// a class path resource pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// all class path resources with the given name
				//路径不包含通配符
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		}
		else {
			// Generally only look for a pattern after a prefix here,
			// and on Tomcat only after the "*/" separator for its "war:" protocol.
			int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
					locationPattern.indexOf(':') + 1);
			//路径包含通配符
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				// a file pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// a single resource with the given name
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}
```

处理逻辑如下图：

![](https://raw.githubusercontent.com/fcdxdx/tuchuang/master/87106504.jpg)



#### findAllClassPathResources()

当 locationPattern 以 `classpath*:` 开头但是不包含通配符，则调用`findAllClassPathResources()` 方法加载资源。该方法返回 classes 路径下和所有 jar 包中的所有相匹配的资源。

```java
protected Resource[] findAllClassPathResources(String location) throws IOException {
        String path = location;
        if (path.startsWith("/")) {
            path = path.substring(1);
        }
        Set<Resource> result = doFindAllClassPathResources(path);
        if (logger.isDebugEnabled()) {
            logger.debug("Resolved classpath location [" + location + "] to resources " + result);
        }
        return result.toArray(new Resource[0]);
    }
```

真正执行加载的是在 `doFindAllClassPathResources()`方法，如下：

```java
 protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {
        Set<Resource> result = new LinkedHashSet<>(16);
        ClassLoader cl = getClassLoader();
        Enumeration<URL> resourceUrls = (cl != null ? cl.getResources(path) : ClassLoader.getSystemResources(path));
        while (resourceUrls.hasMoreElements()) {
            URL url = resourceUrls.nextElement();
            result.add(convertClassLoaderURL(url));
        }
        if ("".equals(path)) {
            addAllClassLoaderJarRoots(cl, result);
        }
        return result;
    }
```

`doFindAllClassPathResources()` 根据 ClassLoader 加载路径下的所有资源。在加载资源过程中如果，在构造 PathMatchingResourcePatternResolver 实例的时候如果传入了 ClassLoader，则调用其 `getResources()`，否则调用`ClassLoader.getSystemResources(path)`。 `ClassLoader.getResources()`如下:

```java
public Enumeration<URL> getResources(String name) throws IOException {
        @SuppressWarnings("unchecked")
        Enumeration<URL>[] tmp = (Enumeration<URL>[]) new Enumeration<?>[2];
        if (parent != null) {
            tmp[0] = parent.getResources(name);
        } else {
            tmp[0] = getBootstrapResources(name);
        }
        tmp[1] = findResources(name);

        return new CompoundEnumeration<>(tmp);
    }
```

看到这里是不是就已经一目了然了？如果当前父类加载器不为 null，则通过父类向上迭代获取资源，否则调用 `getBootstrapResources()`。这里是不是特别熟悉，(*^▽^*)。

若 path 为 空（“”）时，则调用 `addAllClassLoaderJarRoots()`方法。该方法主要是加载路径下得所有 jar 包，方法较长也没有什么实际意义就不贴出来了。

通过上面的分析，我们知道 `findAllClassPathResources()` 其实就是利用 ClassLoader 来加载指定路径下的资源，不过它是在 class 路径下还是在 jar 包中。如果我们传入的路径为空或者 `/`，则会调用 `addAllClassLoaderJarRoots()` 方法加载所有的 jar 包。



#### findAllClassPathResources()

当 locationPattern 以 classpath*: 开头且当中包含了通配符，则调用该方法进行资源加载。如下：

```java
protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
        // 确定跟路径
        String rootDirPath = determineRootDir(locationPattern);
        String subPattern = locationPattern.substring(rootDirPath.length());

        // 获取根据路径下得资源
        Resource[] rootDirResources = getResources(rootDirPath);

        Set<Resource> result = new LinkedHashSet<>(16);
        for (Resource rootDirResource : rootDirResources) {
            rootDirResource = resolveRootDirResource(rootDirResource);
            URL rootDirUrl = rootDirResource.getURL();
            // bundle 资源类型
            if (equinoxResolveMethod != null && rootDirUrl.getProtocol().startsWith("bundle")) {
                URL resolvedUrl = (URL) ReflectionUtils.invokeMethod(equinoxResolveMethod, null, rootDirUrl);
                if (resolvedUrl != null) {
                    rootDirUrl = resolvedUrl;
                }
                rootDirResource = new UrlResource(rootDirUrl);
            }

            // VFS 资源
            if (rootDirUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
                result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirUrl, subPattern, getPathMatcher()));
            }

            // Jar
            else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) {
                result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
            }
            else {
                result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
            }
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Resolved location pattern [" + locationPattern + "] to resources " + result);
        }
        return result.toArray(new Resource[0]);
    }
```

方法有点儿长，但是思路还是很清晰的，主要分两步：

1. 确定目录，获取该目录下得所有资源
2. 在所获得的所有资源中进行迭代匹配获取我们想要的资源。

在这个方法里面我们要关注两个方法，一个是 `determineRootDir()`,一个是 `doFindPathMatchingFileResources()`。

`determineRootDir()`主要是用于确定根路径，如下：

```java
 protected String determineRootDir(String location) {
        int prefixEnd = location.indexOf(':') + 1;
        int rootDirEnd = location.length();
        while (rootDirEnd > prefixEnd && getPathMatcher().isPattern(location.substring(prefixEnd, rootDirEnd))) {
            rootDirEnd = location.lastIndexOf('/', rootDirEnd - 2) + 1;
        }
        if (rootDirEnd == 0) {
            rootDirEnd = prefixEnd;
        }
        return location.substring(0, rootDirEnd);
    }
```

该方法一定要给出一个确定的根目录。该根目录用于确定文件的匹配的起始点，将根目录位置的资源解析为 `java.io.File` 并将其传递到 `retrieveMatchingFiles()`，其余为知用于模式匹配，找出我们所需要的资源。

确定根路径如下:

|               原路径               |      确定根路径       |
| :--------------------------------: | :-------------------: |
| `classpath*:test/cc*/spring-*.xml` |  `classpath*:test/`   |
| `classpath*:test/aa/spring-*.xml`  | `classpath*:test/aa/` |

确定根路径后，则调用 `getResources()` 方法获取该路径下得所有资源，然后迭代资源获取符合条件的资源。

至此 Spring 整个资源记载过程已经分析完毕。下面简要总结下：

- Spring 提供了 Resource 和 ResourceLoader 来统一抽象整个资源及其定位。使得资源与资源的定位有了一个更加清晰的界限，并且提供了合适的 Default 类，使得自定义实现更加方便和清晰。
- DefaultResource 为 Resource 的默认实现，它对 Resource 接口做了一个统一的实现，子类继承该类后只需要覆盖相应的方法即可，同时对于自定义的 Resource 我们也是继承该类。
- DefaultResourceLoader 同样也是 ResourceLoader 的默认实现，在自定 ResourceLoader 的时候我们除了可以继承该类外还可以实现 ProtocolResolver 接口来实现自定资源加载协议。
- DefaultResourceLoader 每次只能返回单一的资源，所以 Spring 针对这个提供了另外一个接口 ResourcePatternResolver ，该接口提供了根据指定的 locationPattern 返回多个资源的策略。其子类 PathMatchingResourcePatternResolver 是一个集大成者的 ResourceLoader ，因为它即实现了 `Resource getResource(String location)` 也实现了 `Resource[] getResources(String locationPattern)`。