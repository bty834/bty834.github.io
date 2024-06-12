---
title: SpringBoot application.yml/.properties配置文件加载过程
categories: [编程, Java ]
tags: [spring,springboot]
---

参考：[Springboot源码之application.yaml读取过程](https://blog.csdn.net/weixin_43843104/article/details/110131011?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164627388216780255276878%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164627388216780255276878&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~baidu_landing_v2~default-2-110131011.nonecase&utm_term=springboot%E8%AF%BB%E5%8F%96application.yml%E8%BF%87%E7%A8%8B&spm=1018.2226.3001.4450)

## 当SpringBoot版本<2.4.0时：

SpringBoot配置文件一般为*application.yml*或*application.properties*等，其加载流程在`SpringApplication`的`run()`方法中的`ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);`中完成，当环境准备好会触发`EventPublishingRunListener implement SpringApplicationRunListener`的 `environmentPrepared()`方法，该方法广播`ApplicationEnvironmentPreparedEvent`，该事件有一个`ConfigFileApplicationListener implements EnvironmentPostProcessor, SmartApplicationListener, Ordered`监听器，该类用于加载配置文件。以上具体的调用过程如下：
![在这里插入图片描述](/assets/2023/06/09/1.png)
具体看看`ConfigFileApplicationListener`这个类如何加载配置文件的：

```java
public class ConfigFileApplicationListener implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {
	...
	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		// 向下转型
		if (event instanceof ApplicationEnvironmentPreparedEvent) {
			onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
		}
		if (event instanceof ApplicationPreparedEvent) {
			onApplicationPreparedEvent(event);
		}
	}

	private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		// 获取所有的EnvironmentPostProcessor对象
		List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
		// this对象也是一个EnvironmentPostProcessor，加入
		postProcessors.add(this);
		AnnotationAwareOrderComparator.sort(postProcessors);
		for (EnvironmentPostProcessor postProcessor : postProcessors) {
		// 遍历进行环境后置处理
		// 当处理到this时，加载文件，会调用this.postProcessEnvironment	
		postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
		}
	}
	...
		@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		addPropertySources(environment, application.getResourceLoader());
	}

	protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
		RandomValuePropertySource.addToEnvironment(environment);
		// 正式开始load
		new Loader(environment, resourceLoader).load();
	}
}

		// 这里用来循环location，点开getSearchLocations()可以看到配置文件的路径搜索顺序(LinkedHashSet是有序的)如下:
		// "file:./config/"
		// "file:./config/*/"
		// "file:./"
		// "classpath:/config/"
		// "classpath:/"
		private void load(Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
			getSearchLocations().forEach((location) -> {
				boolean isDirectory = location.endsWith("/");
				// 用来循环配置文件名称，一般只有一个就是application，除非设置spring.profiles.active项
				Set<String> names = isDirectory ? getSearchNames() : NO_SEARCH_NAMES;
				names.forEach((name) -> load(location, name, profile, filterFactory, consumer));
			});
		}
		
		private void load(String location, String name, Profile profile, DocumentFilterFactory filterFactory,
				DocumentConsumer consumer) {
			if (!StringUtils.hasText(name)) {
				for (PropertySourceLoader loader : this.propertySourceLoaders) {
					if (canLoadFileExtension(loader, location)) {
						load(loader, location, profile, filterFactory.getDocumentFilter(profile), consumer);
						return;
					}
				}
				throw new IllegalStateException("File extension of config file location '" + location
						+ "' is not known to any PropertySourceLoader. If the location is meant to reference "
						+ "a directory, it must end in '/'");
			}
			Set<String> processed = new HashSet<>();
			// PropertySourceLoader有两个实现类：PropertiesPropertySourceLoader和YamlPropertySourceLoader
			// 前者用于搜索properties和xml文件
			// 后者用于搜索yml和yaml文件
			for (PropertySourceLoader loader : this.propertySourceLoaders) {
				// 遍历后缀文件名，
				for (String fileExtension : loader.getFileExtensions()) {
					if (processed.add(fileExtension)) {
						// 这里load
						loadForFileExtension(loader, location + name, "." + fileExtension, profile, filterFactory,
								consumer);
					}
				}
			}
		}
```
## 当SpringBoot版本>=2.4.0时：
上文中`ConfigFileApplicationListener`类于2.4.0版本被废弃，取而代之的是`ConfigDataEnvironmentPostProcessor`，所有的`EnvironmentPostProcessor`通过`EnvironmentPostProcessorApplicationListener`调用，该`ApplicationListener`的顺序Order为`Ordered.HIGHEST_PRECEDENCE + 10`（而bootstrap.yaml等文件的`BootstrapApplicationListener`的顺序Order为`Ordered.HIGHEST_PRECEDENCE + 5`，所以boostrap先于application解析，同样application会覆盖boostrap配置），整个调用链路如下：

`SpringApplication#run` -> `SpringApplication#prepareEnvironment`->`SpringApplicationRunListeners#environmentPrepared` -> 系列`ApplicationListener`按照Order从小到达执行
通过这两个类的注释也可以看出：

![在这里插入图片描述](/assets/2023/06/09/2.png)

![在这里插入图片描述](/assets/2023/06/09/3.png)

之前加载application.yml or properties文件和springboot是耦合在一个ConfigFileApplicationListener中，也就是说，只有调用springboot的这个监听器才能解析配置文件，而现在SpringBoot这里==抽象==出了加载文件的过程。

先来看一下`org.springframework.boot:spring-boot.jar`包下的$META-INF$的`spring.factories`文件，需要注意以下三个key:value：
![在这里插入图片描述](/assets/2023/06/09/4.png)

看到下面两个接口属于同一个包，显然是有一定联系的。看看它们的注释：

> **ConfigDataLocationResolver:**    Strategy interface used to resolve {@link `ConfigDataLocation` locations} into one or more {@link `ConfigDataResource` resources}.
> **ConfigDataLoader:**    Strategy class that can be used to load {@link `ConfigData`} for a given{@link ConfigDataResource}.

他们都是策略模式的上层接口，之间的联系如下图：
![在这里插入图片描述](/assets/2023/06/09/5.png)

再看第一个接口：
> **PropertySourceLoader:**    to load a {@link `PropertySource`}.


实际上，在加载springboot配置文件的过程中`StandardConfigDataSource extends ConfigDataSource`中通过字段`StandardConfigDataReference reference` 的字段方法`PropertySourceLoader.load()`来实现`ConfigDataLoader.load()`。这个`StandardConfigDataReference`既有 `ConfigDataLocation` 也有`PropertySourceLoader`，起到了承上启下的reference作用。结构图如下：
![在这里插入图片描述](/assets/2023/06/09/6.png)

至于StandardConfigDataLoader怎么调用的，我就不画图了，流程如下：
`ConfigDataEnvironmentPostProcessor`的`postProcessEnvironment()`$\rightarrow$`ConfigDataEnvironment`的`processAndApply()`$\rightarrow$同类中的`processInitial()`$\rightarrow$`ConfigDataEnvironmentContributors`的`withProcessedImports()`$\rightarrow$`ConfigDataImporter`的`resolveAndLoad()`$\rightarrow$同类的`load()`$\rightarrow$`ConfigDataLoaders`的`load()`$\rightarrow$`StandardConfigDataLoader`的`load()`。

