# RestTemplate的负载均衡

今天看到项目里使用了RestTemplate实现了负载均衡，如下：

```java
/**
 * 负载均衡
 *
 * @return rest template
 */
@Bean
@LoadBalanced
RestTemplate restTemplate() {
	return new RestTemplate();
}
```

可以看到短短的几行，仅仅加了`@Bean`和`@LoadBalanced`两个注解就轻易的实现了负载均衡，感觉很有意思，所以到网上找了些资料总结如下，特此记录

首先来看下`@LoadBalanced`注解

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```

里面没有定义任何属性，但是值得注意的是它被`@Qualifier`注解修饰，为什么要这样呢，接着向下看

然后去查找是否有相关的自动配置类？结果真的找到一个名为`LoadBalancerAutoConfiguration`的自动配置类。

先来看此类上的注解：

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
```

可以很清楚的看到当存在`RestTemplate`类并且在Spring容器中存在`LoadBalancerClient`类型的bean的时候此配置类才生效。RestTemplate是我们创建的，这没问题。但是这个LoadBalancerClient是什么时候注册到Spring容器中的？翻到源码中发现它是一个接口，紧接着查找它的实现，发现它有一个实现类：`RibbonLoadBalancerClient`，这是Ribbon在客户端的负载均衡的实现，然后发现引用的依赖里面确实包含Ribbon。所以此配置类就生效了。另外说一句可以查看**LoadBalancerRetryProperties**到配置文件中进行相关配置：`spring.cloud.loadbalancer.retry.enabled = true`意为当请求失败时是否重试。

该类中存在一个成员变量：

```java
@LoadBalanced
@Autowired(required = false)
private List<RestTemplate> restTemplates = Collections.emptyList();
```

这个成员变量很重要因为它保存了所有**被@LoadBalanced注解**修饰的RestTemplate的bean，原因是因为`@Autowired`注解不仅可以注入单个实例，还可以导入如List这样的集合，又因为此成员变量同样由`@LoadBalanced`注解修饰，隐含着它也是被@Qualifier修饰，那么它就将所有的**同样由@Qualifier(@LoadBalanced)修饰的RestTemplate实例添加到此成员变量中了**这样就可以理解为什么声明RestTemplate时要加上此注解。如果难理解的话可以参考这篇文章：`https://blog.csdn.net/xiao_jun_0820/article/details/78917215`

接着向下看可以看到下面负载均衡拦截器类里面配置了一个Bean：

```java
@Bean
@ConditionalOnMissingBean
public RestTemplateCustomizer restTemplateCustomizer(
		final LoadBalancerInterceptor loadBalancerInterceptor) {
	return new RestTemplateCustomizer() {
		@Override
		public void customize(RestTemplate restTemplate) {
			List<ClientHttpRequestInterceptor> list = new ArrayList<>(
					restTemplate.getInterceptors());
			list.add(loadBalancerInterceptor);
			restTemplate.setInterceptors(list);
		}
	};
}
```

可以看到在返回的自定义RestTemplate中它将指定的负载均衡的拦截器加入到了RestTemplate的拦截器列表中，这样RestTemplate就会在请求时用到负载均衡的拦截器从而实现负载均衡

```java
@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
		final List<RestTemplateCustomizer> customizers) {
	return new SmartInitializingSingleton() {
		@Override
		public void afterSingletonsInstantiated() {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		}
	};
}
```

这个Bean遍历了restTemplate列表中的每个实例，然后分别调用上面的方法为每个实例加上了负载均衡的拦截器

接下来看看拦截器是怎样工作的，首先根据上面添加拦截器的那部分代码可以看到传进来了一个`LoadBalancerInterceptor`类型的Bean，然后发现自动配置类中也声明了这么一个Bean

```java
@Bean
public LoadBalancerInterceptor ribbonInterceptor(
		LoadBalancerClient loadBalancerClient,
		LoadBalancerRequestFactory requestFactory) {
	return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
}
```

最后返回了一个新的LoadBalancerInterceptor对象，接着进入到LoadBalancerInterceptor中，发现它重写了一个`intercept()`方法

```java
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
		final ClientHttpRequestExecution execution) throws IOException {
	final URI originalUri = request.getURI();
	String serviceName = originalUri.getHost();
	return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
}
```

此方法中先是获取了URL和主机名，进而调用成员变量`loadBalancer`的`execute()`方法。这里loadBalancer的类型为LoadBalancerClient是一个接口，它有一个实现为`RibbonLoadBalancerInterceptor`，所以它最终回去调用这个`RibbonLoadBalancerInterceptor`的`execute()`方法

```java
@Override
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
	ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
	Server server = getServer(loadBalancer);
	if (server == null) {
		throw new IllegalStateException("No instances available for " + serviceId);
	}
	RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
			serviceId), serverIntrospector(serviceId).getMetadata(server));
	return execute(serviceId, ribbonServer, request);
}
```

由于接下来的源码多余复杂，涉及到了Ribbon的源码不太好理解，所以不再细致的介绍。关于后续内容可以参照此博客加深理解：`http://blog.didispace.com/springcloud-sourcecode-ribbon/`，如果有机会未来会把此文章补全