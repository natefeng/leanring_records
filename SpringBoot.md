# SpringBoot

## 底层注解

### Configuration

想当于告诉springBoot该类是一个配置类 == spring以前的配置文件

![image-20210219213008818](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210219213008818.png)

bean注解给容器中添加组件，以方法名为组件的id,返回类型为组件类型

```java
public @interface Configuration {
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";
    //Full模式 生成配置类的代理对象，优先取容器，单例模式
    //Lite(proxyBeanMethods = false)，普通对象，每次取出来的对象不同
    boolean proxyBeanMethods() default true;
    设置为false跳过springBoot检查，启动非常快
    设置为true会进行大量检查
}

```

![image-20210219214429551](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210219214429551.png)

### Import

![image-20210219215417274](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210219215417274.png)

将指定类型的组件导入到容器中

给容器中自动创建出这两个类型的组件，导入的组件默认名字为全类名

### RestController

该注解应该不属于SpringBoot,属于spring4.0新增的

目的是解决方法上标注ResponBody注解和Controller注解   解决冗余注解和直接返回数据麻烦

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(
        annotation = Controller.class
    )
    String value() default "";
}
```

### Conditional

条件装配注解，比如ConditionalBean表明容器中有指定的Bean下面的类或者方法代码才生效

![image-20210220202855751](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210220202855751.png)

### ImportResource

可以指定导入某个配置文件

比如在xml文件中注册了许多Bean，然后可以直接将该注解标注在配置类上让该xml文件生效

![image-20210220203327186](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210220203327186.png)

### ConfigurationProperties

配置绑定

快速将properties里面的值与JAVA Bean 一一绑定起来 前提是该类必须在容器中比如说该类标注了Compoent注解

```java
mycar.brand = BYD
mycar.price = 100000;    
```

![image-20210220204124845](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210220204124845.png)

还有一种方法

在配置类上标注EnableConfigurationProperties，并且指定要为哪个类开启配置绑定功能，还有将该类加入到容器中，常用于第三方JAR包，因为第三方JAR包如果没有标注Compoent注解，没有在容器中可以使用该方法

![image-20210220204731195](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210220204731195.png)

## 自动配置原理

重点在启动配置类上

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

```

先来观察springBootConfiguration

```java
@Configuration
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
//表明标注了该注解的类是一个配置类，也就是启动配置类是一个配置类
```

ComponentScan是一个包扫描，不用说

接下来是EnableAutoConfiguration

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
可以看到用Import导入了一个组件+一个AutoConfigurationPackage注解
    
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
该注解又是一个Import注解，表明导入了Registrar这个组件
```

```java
/**
	 * {@link ImportBeanDefinitionRegistrar} to store the base package from the importing
	 * configuration.
	 */
	static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

		@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
		}

		@Override
		public Set<Object> determineImports(AnnotationMetadata metadata) {
			return Collections.singleton(new PackageImports(metadata));
		}

	}
//容器启动的时候会调用该组件的registerBeanDefinitions方法，传入标注了该AutoConfigurationPackage注解的类的元数据信息，通过该信息拿到包名然后注册到容器

```

![image-20210221123854996](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221123854996.png)

所以说如果没有指定扫描包，默认就是启动配置类所在的包会全部扫描进去

AutoConfigurationImportSelector.class 导入的该类里面的核心方法

> - [ ] ```java
>   @Override
>   	public String[] selectImports(AnnotationMetadata annotationMetadata) {
>   		if (!isEnabled(annotationMetadata)) {
>   			return NO_IMPORTS;
>   		}
>   		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata); //根据注解元数据去获取自动配置实例
>   		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
>   	}
>   
>   
>   
>   	protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
>   		if (!isEnabled(annotationMetadata)) {
>   			return EMPTY_ENTRY;
>   		}
>   		AnnotationAttributes attributes = getAttributes(annotationMetadata);
>   	  List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
>   		configurations = removeDuplicates(configurations);
>   		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
>   		checkExcludedClasses(configurations, exclusions);
>   		configurations.removeAll(exclusions);
>   		configurations = getConfigurationClassFilter().filter(configurations);
>   		fireAutoConfigurationImportEvents(configurations, exclusions);
>   		return new AutoConfigurationEntry(configurations, exclusions);
>   	}
>   ```

```java
 List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
核心方法  获取候选配置，会拿到所有场景的自动配置类名
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader()); //根据spring工厂去加载，传入了一个注解EnableAutoConfiguration，和一个类加载器
		Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}


下面代码就是默认从Meta-info下的文件来读取所有的类
spring-boot-autoconfigure-2.4.3.jar 从该包下的Meta-inf去默认下载所有可能需要的场景，然后根据
条件配置去按需选择需要的场景 这个时候就运用到了大量的条件装配，比如ConditionOnClass(Advice.class)
意思就是如果导入了切面所需要的jar包，该配置类才会生效，截图如下    
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = (Map)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            HashMap result = new HashMap();

            try {
                Enumeration urls = classLoader.getResources("META-INF/spring.factories");
```

```java
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.r2dbc.R2dbcDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.r2dbc.R2dbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration,\
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration,\
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration,\
org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration,\
org.springframework.boot.autoconfigure.influx.InfluxDbAutoConfiguration,\
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration,\
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration,\
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\
org.springframework.boot.autoconfigure.jooq.JooqAutoConfiguration,\
org.springframework.boot.autoconfigure.jsonb.JsonbAutoConfiguration,\
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration,\
org.springframework.boot.autoconfigure.availability.ApplicationAvailabilityAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.embedded.EmbeddedLdapAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration,\
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\
org.springframework.boot.autoconfigure.neo4j.Neo4jAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.quartz.QuartzAutoConfiguration,\
org.springframework.boot.autoconfigure.r2dbc.R2dbcAutoConfiguration,\
org.springframework.boot.autoconfigure.r2dbc.R2dbcTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketRequesterAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketServerAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketStrategiesAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.rsocket.RSocketSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.saml2.Saml2RelyingPartyAutoConfiguration,\
org.springframework.boot.autoconfigure.sendgrid.SendGridAutoConfiguration,\
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.reactive.ReactiveOAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.resource.reactive.ReactiveOAuth2ResourceServerAutoConfiguration,\
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\
org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration,\
org.springframework.boot.autoconfigure.task.TaskSchedulingAutoConfiguration,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration,\
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration,\
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.HttpHandlerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.error.ErrorWebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.function.client.ClientHttpConnectorAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.function.client.WebClientAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.client.WebServiceTemplateAutoConfiguration
```

![image-20210221132839437](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221132839437.png)

总结：

- SpringBoot会加载所有的自动配置类 xxxAutoConfiguration

- 每个自动配置类按照条件进行生效，默认会绑定配置文件指定的值，xxxxProperties里面拿，xxxProperties和配置文件进行了绑定

- 生效的自动配置类就会给容器中装配很多组件

- 只要容器中有这些组件，相当于这些功能就有了

- 定制化配置 

    用户直接可以使用@bean替换底层的组件

    用户去看这个组件是获取的配置文件什么值然后去修改、

  

  

## 简化开发

### lombok

![image-20210221153255024](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221153255024.png)

lombok简化开发，编译后生成get,set方法等等

并且可以标注@Slf4j注解。使用日志来打印信息

### Devtools 

```java
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
```

热更，ctrl+F9，本质上还是重新编译重启而已，

### Spring Initailizr(项目初始化向导)

懒人必备，已经搭建好环境可以直接进行业务逻辑的开发

## YML

标记语言，非常适合用来以数据为中心的配置文件

### 基本语法

- key: value； kv之间有空格 :后面存在空格

- 大小写敏感

- 使用缩进表示层级关系

- 缩进不允许使用tab，只允许空格

- ‘#’表示注释

  

![image-20210221160112717](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221160112717.png)

![image-20210221161013391](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221161013391.png)

![image-20210221161444764](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221161444764.png)

上图为表示一个复杂对象的写法，yml语法格式

并且yml里面使用双引号会转义特殊字符比如\n 会换行,单引号就会单纯输出为字符串，不会转义特殊字符

## 核心功能

### 静态资源访问

类路径下：/static(or /public or /resources or /META-INF/resources)

这些路径下都可以访问到静态资源，官方文档里面解释的

默认是找/**，也就是默认拦截所有请求，先去看Controller能不能处理，不能处理交给静态资源处理器

```yaml
spring:
  mvc:
    static-path-pattern: /resources/**
spring:
  mvc:
    static-path-pattern: /resources/**
  web:
    resources:
      static-locations: classpath:/haha/   //改变静态资源的访问路径，已经过时
```

默认是/**，可以更改为上面代码的路径，因为可能会重名优先去找动态请求

![image-20210221165720026](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210221165720026.png)

### 欢迎页支持

- 静态资源路径下 index.html
  - 可以配置静态资源路径
  - 不可以配置静态资源的访问前缀，否则导致Index.html不能被默认访问

### 静态资源源码分析

WebMvcAutoConfiguration

```java

public WebMvcAutoConfigurationAdapter(WebProperties webProperties, WebMvcProperties mvcProperties,
				ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
				ObjectProvider<DispatcherServletPath> dispatcherServletPath,
				ObjectProvider<ServletRegistrationBean<?>> servletRegistrations) {
			this.mvcProperties = mvcProperties;
			this.beanFactory = beanFactory;
			this.messageConvertersProvider = messageConvertersProvider;
			this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
			this.dispatcherServletPath = dispatcherServletPath;
			this.servletRegistrations = servletRegistrations;
			this.mvcProperties.checkConfiguration();
		}
```

```java
//处理WebJars请求和默认静态资源访问路径 this.resourceProperties.getStaticLocations()
@Override
		protected void addResourceHandlers(ResourceHandlerRegistry registry) {
			super.addResourceHandlers(registry);
			if (!this.resourceProperties.isAddMappings()) {
				logger.debug("Default resource handling disabled");
				return;
			}
			ServletContext servletContext = getServletContext();
			addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
			addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
				registration.addResourceLocations(this.resourceProperties.getStaticLocations());
				if (servletContext != null) {
					registration.addResourceLocations(new ServletContextResource(servletContext, SERVLET_LOCATION));
				}
			});
		}
```

### Rest映射以及源码解析

![image-20210222145053310](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222145053310.png)

想要使用Rest风格

核心就是使用Http请求还对资源进行操作，相同路径，使用GET,PUT,POST,DELETE来完成对同一资源的不同操作

但是由于form表单只支持get,post操作

![image-20210222145223984](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222145223984.png)

![image-20210222145431677](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222145431677.png)

约定大于配置，HiddenHttpMethodFilter来解决表单不能提交put和delete的请求

```java
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        HttpServletRequest requestToUse = request;
        if ("POST".equals(request.getMethod()) && request.getAttribute("javax.servlet.error.exception") == null) {
            String paramValue = request.getParameter(this.methodParam);
            if (StringUtils.hasLength(paramValue)) {
                String method = paramValue.toUpperCase(Locale.ENGLISH);
                if (ALLOWED_METHODS.contains(method)) {
                    requestToUse = new HiddenHttpMethodFilter.HttpMethodRequestWrapper(request, method);
                }
            }
        }

        filterChain.doFilter((ServletRequest)requestToUse, response);
    }
```

```
@Bean
@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)
public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
   return new OrderedHiddenHttpMethodFilter();
}
可以看到默认是关闭的，所以需要去配置文件去开启HiddenHttpMethodFilter功能
```

Rest原理(表单提交要使用Rest的时候)

- 表单提交要带上_method=PUT

- 请求过来被**HiddenHttpMethodFilter**拦截

  - 请求是否正常，并且是**POST**

    - 获取到**_method**的值

    - 兼容以下请求：**PUT,DELETE,PATCH**

    - 原生request(post)，包装模式requestWrapper重写了getMethod()方法，返回的是传入的值

    - 过滤链放行的时候用wrapper，以后方法调用getMethod()都用requestWrapper的

      

Rest使用客户端工具不需要使用过滤器，原因是表单提交只有get和post

- postman模拟
- 安卓直接发put,delete请求

### 请求映射原理

SpringMvc所有的功能分析都在于doDispatch方法里面

![image-20210222153222840](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222153222840.png)

```java
mappedHandler = getHandler(processedRequest); 该方法用于找出哪个方法可以处理前端发来的请求
```

```java
//遍历每个handlerMapping。找到可以处理该请求的handler	
@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

![image-20210222154510468](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222154510468.png)

所有的HandlerMapping

首先看欢迎页，上面已经提到过

RequestMappingHandlerMapping：保存了@RequestMapping注解和Handler的映射规则

![image-20210222154908803](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222154908803.png)

在映射注册中心保存了所有的映射，比如/bug.jpg 是由哪个类的哪个方法

### 常用参数注解使用

![image-20210222202759905](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222202759905.png)

#### PathVariable

![image-20210222162259015](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222162259015.png)

**PathVariable**可以读取路径上的变量，比如前端发 /car/1/owner/fengjiahao/

可以读取到1和fengjiahao变量，并且存入pv之中

#### RequestHeader

**RequestHeader**获取请求头的一个参数

![image-20210222162816214](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222162816214.png)

获取所有请求头参数![image-20210222162907710](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222162907710.png)

#### RequestParam



@RequestParam获取请求参数

![image-20210222163033683](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222163033683.png)

![image-20210222163109239](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222163109239.png)

获取单个直接写参数，获取多个比如篮球和游戏，直接写集合

#### CookieValue

@CookieValue 获取cookie的值

![image-20210222163950837](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222163950837.png)

![image-20210222164110915](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222164110915.png)

可以拿整个cookie对象

#### RequestBody

获取到整个表单的数据

![image-20210222164550102](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222164550102.png)

![image-20210222164557309](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222164557309.png)

![image-20210222164603769](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222164603769.png)

相当于就是将表单的数据映射到参数上

#### RequestAttribute

![image-20210222165051566](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222165051566.png)

获取请求域中的属性，由于是转发，属于同一个请求，所以可以直接使用该注解取出请求中的属性

也可以使用HttpServletRequest请求来获取属性

#### MatrixVariable 矩阵变量

![image-20210222171950764](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222171950764.png)

矩阵变量，以:结尾。上图比如 boss下的1并且年龄等于20的，和2年龄等于20的

low=34万的，品牌在byd,audi,yd这些里面选

SpringBoot默认关闭矩阵变量功能

以下两种开启办法

默认UrlPathHelper的参数RemoveSemicolonContent为true，也就是移除

![image-20210222173322032](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222173322032.png)

![image-20210222173916249](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222173916249.png)

![image-20210222173946180](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222173946180.png)

想要访问到不能直接写sell，必须使用{path}方式

![image-20210222174339143](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222174339143.png)

当同时需要获取名的矩阵变量参数，需要指定pathVar，指定是哪个路径下的变量

### 各种类型参数解析原理

```
// Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

**相当于是对于上面找到处理请求的方法进行参数封装和反射调用**

![image-20210222175750513](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222175750513.png)

如何找适合的adapter?也是遍历所有的handlerAdapter，去寻找适合的

```java
@Override
public final boolean supports(Object handler) {
   return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}
```

只要是HandlerMethod就可以，如果返回true则直接返回当前adapter,一般都是RequestMappingHandlerAdapter

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   if (this.handlerAdapters != null) {
      for (HandlerAdapter adapter : this.handlerAdapters) {
         if (adapter.supports(handler)) {
            return adapter;
         }
      }
   }
   throw new ServletException("No adapter for handler [" + handler +
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

```
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler()); //执行处理器
```

```
mav = invokeHandlerMethod(request, response, handlerMethod); //真正执行可以处理请求的方法
```

![image-20210222181815151](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222181815151.png)

```
ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
if (this.argumentResolvers != null) {
   invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
}
```

提供**参数解析器**，可以解析各种参数类型，比如标注了RequestParam的，PathVariable的等等

合计26个可以解析的类型

里面的设计

![image-20210222182043476](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222182043476.png)

首先调用supports将参数传入，判断是否支持，如果支持则解析

还有**返回值处理器**

![image-20210222182141289](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222182141289.png)

15种可以返回的类型，比如ResponBody的，modelAndView类型的等等

```
invocableMethod.invokeAndHandle(webRequest, mavContainer); //此处是执行方法的
```

```
Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs); //此处是真正执行可以处理请求的方法的，并且还会有返回值
```

```java
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    Object[] args = this.getMethodArgumentValues(request, mavContainer, providedArgs); //确定方法参数值，绑定参数值
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }

    return this.doInvoke(args); //反射执行目标方法
}
```

```java
//循环遍历26个参数解析器，找出可以解析该参数的，比如是否是被RequestParam注解标注了呢等等
@Nullable
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = (HandlerMethodArgumentResolver)this.argumentResolverCache.get(parameter);
    if (result == null) {
        Iterator var3 = this.argumentResolvers.iterator();

        while(var3.hasNext()) {
            HandlerMethodArgumentResolver resolver = (HandlerMethodArgumentResolver)var3.next();
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, resolver);
                break;
            }
        }
    }

    return result;
}
```

```java
//解析参数 ，先找出对应的解析器，然后去解析参数
@Override
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
   if (resolver == null) {
      throw new IllegalArgumentException("Unsupported parameter type [" +
            parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
   }
   return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

### Servlet API参数解析原理

![image-20210222202832639](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222202832639.png)

```java
==== ServletRequestMethodArgumentResolver====
@Override //可以看到，该参数解析器是解析各种Servlet API参数的
public boolean supportsParameter(MethodParameter parameter) {
   Class<?> paramType = parameter.getParameterType();
   return (WebRequest.class.isAssignableFrom(paramType) ||
         ServletRequest.class.isAssignableFrom(paramType) ||
         MultipartRequest.class.isAssignableFrom(paramType) ||
         HttpSession.class.isAssignableFrom(paramType) ||
         (pushBuilder != null && pushBuilder.isAssignableFrom(paramType)) ||
         (Principal.class.isAssignableFrom(paramType) && !parameter.hasParameterAnnotations()) ||
         InputStream.class.isAssignableFrom(paramType) ||
         Reader.class.isAssignableFrom(paramType) ||
         HttpMethod.class == paramType ||
         Locale.class == paramType ||
         TimeZone.class == paramType ||
         ZoneId.class == paramType);
}
```

上图可以看到，解析Servlet API参数也是用的参数解析器，

### 复杂参数原理解析

![image-20210222203844602](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222203844602.png)

比如map,Model(默认里面的数据会被放在request的请求域 request.setAttribute)

RedirectAttributes(重定向携带数据) ServletResponse(response)，等等

```
private final ModelMap defaultModel = new BindingAwareModelMap();
```

向Map里面放数据的时候，解析器本质其实会 getMode，也就是获取BindingAwareModelMap,然后向该类里面放入值

同理Model的解析器也是一样 

都是调用了mavContainer.getModel();返回了**BindingAwareModelMap**进行数据绑定

![image-20210222210404666](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222210404666.png)

Map和Model参数都是向该红色处放入值

```
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException); //处理派发结果 想打关于就是执行完目标方法后需要干的事情，比如说跳转页面或者将数据放入到请求域中等或者进行转发跳转等等
```

![image-20210222211317050](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222211317050.png)

```
render(mv, request, response); //相当于就是执行后续视图解析 参数封装等
```

```java
@Override
public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
      HttpServletResponse response) throws Exception {

   if (logger.isDebugEnabled()) {
      logger.debug("View " + formatViewName() +
            ", model " + (model != null ? model : Collections.emptyMap()) +
            (this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
   }

   Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
   prepareResponse(request, response);
   renderMergedOutputModel(mergedModel, getRequestToExpose(request), response); //执行合并并且输出模型
}
```

```
exposeModelAsRequestAttributes(model, request); //该操作将参数MAP,Model中的数据封装到请求域
```

数据绑定

### 自定义参数绑定原理

```
ServletModelAttributeMethodProcessor
```

该类来解析POJO对象的封装，比如直接将表达封装为Person对象，并且Person对象里面还有级联属性

![image-20210222213551303](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222213551303.png)

![image-20210222215741428](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210222215741428.png)

Http参数万物皆为文本，但是要封装各种类型的参数，所以必须要将参数转换String-》Number等等

```
WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
```

绑定自定义数据的

GenericConversionService类实现了所有的数据转换功能

![image-20210223114731866](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223114731866.png)

![image-20210223115256097](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223115256097.png)

可以定制化MVC功能，比如重写URL路径，和自定义converts，转换器

## 响应处理

### ReturnValueHandle原理

![image-20210223120936363](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223120936363.png)

也是使用的返回值解析器

```java
@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
	invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
                //设置的返回值处理器
			}
```

![image-20210223121859244](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223121859244.png)

处理返回值的一个顶级接口，只有两个方法，逻辑也和参数解析器一样，循环遍历是否支持该返回类型，如果支持，则调用HandleReturnValue处理

RequestResponseBodyMethodProcessor 可以处理返回值标了@ResponseBody注解的。

1.利用MessageConverters进行处理将数据写为Json

- 内容协商(浏览器默认会以请求头的方式告诉服务器他能接受什么样的内容类型)
- 服务器最终会根据自己的能力，决定服务器能生成出什么样内容类型的数据
- SpringMvc会循环遍历容器底层所有的**HttpMessageConverter**

![image-20210223143056727](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223143056727.png)

消息转换其，就是是否支持将此Class类型的对象转换为MediaType

例子 ：Person对象转换为Json

![image-20210223143230988](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223143230988.png)

不同的MessageConverters可以处理不同的数据

![image-20210223144250217](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223144250217.png)

MappingJackson2HttpMessageConverter可以将任何数据转为json，条件判断的时候总是返回true

![image-20210223144338226](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223144338226.png)

### 内容协商

根据客户端接受能力不同，返回不同媒体数据的类型

![image-20210223151524063](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223151524063.png)

用户可以根据请求头的accept 可以接受的数据 服务器动态的将数据进行变化返回给浏览器

**PostMan非常容易更改请求头，但是浏览器默认优先接受什么那么就是什么，为了方便内容协商，开启基于请求参数的内容协商**

也就是开启浏览器参数方式内容协商

![image-20210223152008213](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223152008213.png)

在springBoot配置文件中开启内容协商

![image-20210223152110060](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223152110060.png)

![image-20210223152128304](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223152128304.png)

在发送请求的时候选择自己的format格式，会优先将数据改变为响应的格式

默认只有一个请求头内容协商策略，但是如果开启了请求参数内容协商

那么会新增一个策略

![image-20210223152312483](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223152312483.png)

也就是参数内容协商策略![image-20210223153549977](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223153549977.png)

需求，三种客服端分别响应不同的内容

SpringMVC

都要向配置类添加WebMvcConfigur

如何以参数的方式进行内容协商

比如请求带 format参数来进行不同的内容协商

```java
public interface WebMvcConfigurer {

   /**
    * Help with configuring {@link HandlerMapping} path matching options such as
    * whether to use parsed {@code PathPatterns} or String pattern matching
    * with {@code PathMatcher}, whether to match trailing slashes, and more.
    * @since 4.0.3
    * @see PathMatchConfigurer
    */
   default void configurePathMatch(PathMatchConfigurer configurer) {
   }

   /**
    * Configure content negotiation options.
    */
   default void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
   } //提供了配置内容协商的

   /**
    * Configure asynchronous request handling options.
    */
   default void configureAsyncSupport(AsyncSupportConfigurer configurer) {
   }

   /**
    * Configure a handler to delegate unhandled requests by forwarding to the
    * Servlet container's "default" servlet. A common use case for this is when
    * the {@link DispatcherServlet} is mapped to "/" thus overriding the
    * Servlet container's default handling of static resources.
    */
   default void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
   }
```

## 视图解析与模板引擎

![image-20210223172216279](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223172216279.png)

视图解析器会按照返回的不同规则得到不同的视图

## 拦截器

```
HandlerInterceptor //拦截器的类
```

![image-20210223172606179](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223172606179.png)

preHandle是在处理请求之前，postHandle是在请求处理之后页面跳转之前，

afterCompletion是在页面渲染完成之后

![image-20210223173138545](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223173138545.png)

拦截 与MVC打交道首先应该想到的是WebMvcConfigurer

上面的情况会把静态资源也拦截住

```
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
   return;
} //preHandle方法源码，在ha.handle之前执行，也就解释了为什么是在执行目标方法之前运行
```

#### 拦截器原理

根据当前请求，找到HandleExecutionChain

先来顺序执行所有拦截的preHandle方法

- 如果返回为true，则执行下一个拦截器的PreHandle，因为是执行链，多个拦截器
- 如果false，则会倒序遍历所有的拦截器，直接执行afterCompletion方法

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex) {
   for (int i = this.interceptorIndex; i >= 0; i--) {
      HandlerInterceptor interceptor = this.interceptorList.get(i);
      try {
         interceptor.afterCompletion(request, response, this.handler, ex);
      }
      catch (Throwable ex2) {
         logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
      }
   }
}
```

![image-20210223181625854](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223181625854.png)

1-2-3

如果走到3返回false

那么会执行 2-1 的afterCompletion方法

```java
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
```

上图是执行postHandle的地方，倒序遍历

![image-20210223182634632](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223182634632.png)

前面的步骤有任何异常都会直接触发aftercompletion

```java
if (mappedHandler != null) {
   // Exception (if any) is already handled..
   mappedHandler.triggerAfterCompletion(request, response, null);
} //页面渲染完成，又会倒序遍历aftercompletion方法
```

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex) {
   for (int i = this.interceptorIndex; i >= 0; i--) {
      HandlerInterceptor interceptor = this.interceptorList.get(i);
      try {
         interceptor.afterCompletion(request, response, this.handler, ex);
      }
      catch (Throwable ex2) {
         logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
      }
   }
}
```

![image-20210223183241172](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223183241172.png)

## 文件上传

![image-20210223204555952](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223204555952.png)

文件上传的时候使用RequestPart+MultipartFile来获取form表单提交的照片

![image-20210223210946089](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223210946089.png)

如果使用多文件上传的话 指定multpairt

![image-20210223213920816](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223213920816.png)

源码地方进行判断是不是multipart类型

![image-20210223214132806](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210223214132806.png)

## 错误处理

### 默认规则

- 默认情况下，Spring Boot 提供 /error处理所有错误的映射
- 对于机器客户端，它将生成JSON响应，比如POSTMAN,对于浏览器客户端，响应一个白色页面，两个包含的信息基本相同，错误信息，Http状态和异常消息的详细信息等

要对其进行自定义，添加View解析为Error

### 定制错误处理逻辑

- 自定义错误页
  - error/404.html error/5xx.html springBoot会默认加载error目录下的页面

- @ControllerAdvice+@ExpectionHandle处理异常
- 实现HandleExpectionResolver异常

![image-20210224145330816](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224145330816.png)

也可以看到静态资源目录下

### 源码分析

![image-20210224150456110](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224150456110.png)

根据BasicErrorController请求去找/error路径的请求

给浏览器响应的白页是写死在代码里面的StaticView

![image-20210224150640357](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224150640357.png)

![image-20210224153525005](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224153525005.png)

如果没有任何组件可以处理异常，那么默认会再次发送一次/error请求，然后通过ErroController处理

然后解析错误视图

### 几种异常处理原理

![image-20210224154245671](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224154245671.png)

使用ControllerAdvice注解+ExceptionHandler注解来处理想要处理的异常和想要去的页面

![image-20210224154920894](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224154920894.png)

使用ResponseStatus注解+自定义异常处理

![image-20210224155024128](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224155024128.png)

会自动解析	

![image-20210224155322242](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224155322242.png)

还有自定义一个处理器异常解析器，重写里面的resolveException方法，返回模型和视图

Order时指向优先级，数字越小优先级越高

## 原生组件注入

想要使用原生Servlet功能主要在启动配置类上标注注解

![image-20210224155835273](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224155835273.png)

![image-20210224155938179](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224155938179.png)

还需要WebServlet注解 相当于就成功声明了一个Servlet

效果：直接响应，没有被Spring的拦截器拦截

![image-20210224160119129](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224160119129.png)

单*是Serlvet的写法，双 * 是spring的写法

![image-20210224160335574](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224160335574.png)

![image-20210224161146968](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224161146968.png)

注入原生组件的第二种方法

使用配置类+RegistrationBean系列来加入原生组件

![image-20210224162514014](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224162514014.png)

根据优先匹配原则来匹配

映射的是/路径

## 嵌入式Servlet容器

![image-20210224165235237](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224165235237.png)

启动的时候寻找ServletWebServerFactory

## 定制化原理

场景starter-xxxAutoConfiguration-导入xxx组件-绑定xxxProperties--绑定配置文件项

![image-20210224171307623](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224171307623.png)

![image-20210224173412547](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224173412547.png)

![image-20210224201029825](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224201029825.png)

## 数据访问

### 数据库场景的自动配置

![image-20210224212913814](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224212913814.png)

![image-20210224212922806](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224212922806.png)

![image-20210224213850632](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224213850632.png)

![image-20210224214652621](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210224214652621.png)

### 数据源Druid配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/socket?useUnicode=true&useSSL=true&serverTimezone=UTC
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
```

```xml
 <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.17</version>
 </dependency>
```

```java
@Configuration
public class MyConfig {

  @ConfigurationProperties("spring.datasource")
  public DataSource druidDataSource(){ //表示datasource和DataSource的资源一一绑定
    DruidDataSource druidDataSource = new DruidDataSource();

        return druidDataSource;
  }
  @Bean //表示开启监控页功能
  public ServletRegistrationBean servletRegistrationBean(){
      ServletRegistrationBean<StatViewServlet> registrationBean = new ServletRegistrationBean<>(new StatViewServlet(),"/druid/*");
          return registrationBean;
  }

}
```

以上的方式是使用自定义方式整合Druid

### Starter方式整合Druid

```xml
<dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.17</version>
</dependency>
```

```java
@Import({DruidSpringAopConfiguration.class, DruidStatViewServletConfiguration.class, DruidWebStatFilterConfiguration.class, DruidFilterConfiguration.class})
```

DruidSpringAopConfiguration.class 是用来开启Spring监控的 配置项：**spring.datasource.druid.aop-patterns**

DruidStatViewServletConfiguration.class 是用来开启监控页的 **spring.datasource.druid.stat-view-servlet；**

 DruidWebStatFilterConfiguration.**class**, web监控配置；**spring.datasource.druid.web-stat-filter；默认开启**

DruidFilterConfiguration.**class**}) 所有Druid自己filter的配置

```java
    private static final String FILTER_STAT_PREFIX = "spring.datasource.druid.filter.stat";
    private static final String FILTER_CONFIG_PREFIX = "spring.datasource.druid.filter.config";
    private static final String FILTER_ENCODING_PREFIX = "spring.datasource.druid.filter.encoding";
    private static final String FILTER_SLF4J_PREFIX = "spring.datasource.druid.filter.slf4j";
    private static final String FILTER_LOG4J_PREFIX = "spring.datasource.druid.filter.log4j";
    private static final String FILTER_LOG4J2_PREFIX = "spring.datasource.druid.filter.log4j2";
    private static final String FILTER_COMMONS_LOG_PREFIX = "spring.datasource.druid.filter.commons-log";
    private static final String FILTER_WALL_PREFIX = "spring.datasource.druid.filter.wall";
```

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db_account
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver

    druid:
      aop-patterns: com.atguigu.admin.*  #监控SpringBean
      filters: stat,wall     # 底层开启功能，stat（sql监控），wall（防火墙）

      stat-view-servlet:   # 配置监控页功能
        enabled: true
        login-username: admin
        login-password: admin
        resetEnable: false

      web-stat-filter:  # 监控web
        enabled: true
        urlPattern: /*
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'


      filter:
        stat:    # 对上面filters里面的stat的详细配置
          slow-sql-millis: 1000
          logSlowSql: true
          enabled: true
        wall:
          enabled: true
          config:
            drop-table-allow: false
```

### 整合Mybatis

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.4</version>
</dependency>
```

SqlSessionFactory

SqlSession

Mapper

全局配置文件

mapper映射文件

```java
    //SqlSessionFactory已经配置好
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        
        
public class SqlSessionTemplate implements SqlSession, DisposableBean {
    private final SqlSessionFactory sqlSessionFactory;
    private final ExecutorType executorType;
    private final SqlSession sqlSessionProxy; //SqlSession已经配置好
    
    
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ExecutorType executorType = this.properties.getExecutorType();
        return executorType != null ? new SqlSessionTemplate(sqlSessionFactory, executorType) : new SqlSessionTemplate(sqlSessionFactory);
        
```

以上都已经自动配置好比如SqlSessionFactory和SqlSession

```java
  @GetMapping("/getDept")
    public Dept getDept(@RequestParam("id") int id){
        Dept dept = service.getDept(id);
        return dept;
    }
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
        <!DOCTYPE mapper
                PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
                "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.DeptMapper">
    <select id="getDept" resultType="com.example.demo.pojo.Dept">
        select * from  dept where  id=#{id}
    </select>
</mapper>
```

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true

  mapper-locations: classpath:mybatis/mapper/*.xml # 指定映射路径
```

```java
@Mapper
public interface DeptMapper {

    public Dept getDept(int id);
}
```

以上操作，然后就可以开始和平时一样正常开发了

还有注解版Mybatis,不需要写mapper.xm文件，而是直接在方法上标注@Select,@Insert等注解，不推荐使用

![image-20210225175825046](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210225175825046.png)

@Options注解用于Insert的配置，主键回填，将新增的id赋值给插入进来的city属性

### NoSQL整合

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

自动配置：

- RedisAutoConfiguration 自动配置类。RedisProperties 属性类 --> **spring.redis.xxx是对redis的配置**
- 连接工厂是准备好的。**Lettuce**ConnectionConfiguration、**Jedis**ConnectionConfiguration
- **自动注入了RedisTemplate**<**Object**, **Object**> ： xxxTemplate；
- **自动注入了StringRedisTemplate；k：v都是String**
- **key：value**
- **底层只要我们使用** StringRedisTemplate、RedisTemplate就可以操作redis

## 2、RedisTemplate与Lettuce

```java
    @Test
    void testRedis(){
        ValueOperations<String, String> operations = redisTemplate.opsForValue();

        operations.set("hello","world");

        String hello = operations.get("hello");
        System.out.println(hello);
    }
```

## 单元测试

### Junit5常用注解

#### DisplayName

![image-20210226165225612](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226165225612.png)



![image-20210226165229140](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226165229140.png)

向方法或者类添加名称

#### BeforeEach

@BeforeEach，在每个测试方法运行之前运行

![image-20210226165352210](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226165352210.png)

#### AfterEach

每个测试方法结束之后运行

#### BeforeAll

在所有测试方法运行之前运行必须是static修饰

#### AfterAll

在所有测试方法运行之后运行也必须是static

#### Disabled

禁用掉某个测试方法

#### TimeOut

认为多少秒之后就超时，超时会出现异常

#### SpringBootTest

表示Junit与SpringBoot进行整合，可以使用SpringBoot提供的功能，比如自动注入，事务回滚

### 断言机制



![image-20210226171114464](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226171114464.png)

用于判断

![image-20210226171126641](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226171126641.png)在Assertions包下

![image-20210226172041730](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226172041730.png)

测试前置条件

![image-20210226172729131](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226172729131.png)

嵌套断言

内层的Test可以驱动外层的Test

外层的Test不可以驱动内层的Test

### 参数化测试

![image-20210226174034760](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226174034760.png)

参数化测试所需要

@ParameterizedTest

## 指标监控

![image-20210226174524864](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226174524864.png)

![image-20210226180537796](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226180537796.png)

在actuator后面跟的路径称为端点，默认Web情况下只开启headlth和info,如果想要全部开启需要在配置文件中managerment开启

![image-20210226180915668](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226180915668.png)

![image-20210226180939174](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226180939174.png) 

### Actuator Endpoint

beans,

caches

conditions

![image-20210226181513348](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226181513348.png)

![image-20210226181530514](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226181530514.png)

- health 健康状况
- Metrics 运行时指标
- Loggers 日志记录

![image-20210226203539061](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226203539061.png)

定制断电Endpoint

![image-20210226205009193](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226205009193.png)

需要继承AbstractHealthIndicator

![image-20210226210922177](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226210922177.png)

自定义info信息，可以自己编写yaml文件

![image-20210226212945685](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226212945685.png)

自定义metirex指标信息

![image-20210226213350137](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210226213350137.png)

自定义端点

## 高级特性

### Profile环境切换

用于生产，测试等环境的切换

![image-20210228164752658](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228164752658.png)

![image-20210228164757643](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228164757643.png)

需要使用哪个配置文件就指定那个，式必须为application-xxx-yaml

![image-20210228164946694](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228164946694.png)

![image-20210228165110792](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228165110792.png)

也可以使用这种配置环境

![image-20210228171026537](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228171026537.png)

不同的环境使用不同的实例

![image-20210228171126515](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228171126515.png)

可以不同的环境

标在方法上指定的方法才生效

### 外部化配置

![image-20210228172846273](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228172846273.png)

![image-20210228173146795](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210228173146795.png)

### SpringBoot创建初始化流程

```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		this.bootstrappers = new ArrayList<>(getSpringFactoriesInstances(Bootstrapper.class));
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

- 首先创建SpringApplication类，保存启动类
- 判断当前类型是普通类型或者SERVLET类型或者是响应式编程
- 寻找META-INF/META-INF/spring.factories下的**BootStrapper**类，用于在容器初始化的时候运行,初始化引导器
- 寻找META-INF/META-INF/spring.factories下的**ApplicationContextInitializer**类，用于在容器初始化的时候运行
- 寻找META-INF/META-INF/spring.factories下的**ApplicationListener**类，用于在容器初始化的时候运行
- 确定主程序类

```java
private Class<?> deduceMainApplicationClass() {
		try {
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {
				if ("main".equals(stackTraceElement.getMethodName())) {
					return Class.forName(stackTraceElement.getClassName());
				}
			}
		}
```

### SpringBoot完整启动流程

- StopWatch记录应用的启动时间

- 1. 创建引导上下文(Context环境) ***createBootstrapContext***()

  - 获取到之前的所有**bootstrappers**挨个执行initalize()方法来完成对引导启动器上下文环境设置

- 获取所有的**SpringApplicationRunListeners**
  - 也是去META-INF/spring.factories去找SpringApplicationRunListeners

- 遍历调用**SpringApplicationRunListeners**.start方法

- 准备环境prepareEnvironment()；
- 创建ioc容器(createApplicationContext()) 根据应用类型创建不同的IOC容器
  - Servlet会创建 **AnnotationConfigServletWebServerApplicationContext**

- 准备Application IOC容器的信息 prepareContext()
  - 保存环境信息
  - IOC容器的后置处理流程
  - 应用初始化器 applyInitializers
    - 遍历所有的 **ApplicationContextInitializer** 调用 **initialize**方法。来对IOC容器进行初始化扩展功能
    - 遍历所有的 listener调用 **contextPrepared**。

- - 所有的监听器调用**contextLoaded**。

- 刷新IOC容器  **refreshContext**(context); 进行真正的Spring初始化工作
  - 创建所有组件

- 所有的监听器调用Listeners.started(context)；通知所有的监听器 started
- 调用所有的runners; callRunners()
  - 获取容器中的**ApplicationRunner**
  - 获取容器中的**CommandLineRunner**
  - 合并所有**runner**并且按照order排序
  - 遍历所有的Runner调用**run**方法