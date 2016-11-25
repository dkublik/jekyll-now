---
layout: post
title: Under The Boot
comments: true
---

Remember the times when we had to register dispatchers, viewResolvers, etc. to make our spring application web-app? Then there was _@EnableWebMvc_ annotation and now even this is redundant.  
These days the only thing you need to do is to add _'org.springframework.boot:spring-boot-starter-web'_ dependency to your project and everything else is done automagically.



The same goes for a database connection. Not that long ago minimum db-aware spring-context configuration was:

+ register data source (_&lt;jdbc:embedded-database id="dataSource" type="HSQL"/&gt;_)
+ register entity manager (through entity manager factory) (_&lt;bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean"&gt;_)
+ register transaction manager (_&lt;bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager" &gt;_)
+ turn on annotation driven transaction boundaries (_&lt;tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="false"/&gt;_)

Along the way we dropped xml configs in favour of configurations. Now - everything you need to do is to add another dependency _'org.springframework.boot:spring-boot-starter-data-jpa'_ and some db driver (like _'com.h2database:h2'_) and again - spring creates everything behind the curtains.



I don't know about you - but I grow suspicious when that many things happen without my knowledge. After having been using Spring Boot for a while I needed to look under the hood to feel safe again - not that much under - just enough to get back to my comfort zone. 



#### High Level View

The basic mechanism goes like this:  
All magically appearing beans are registered with spring configurations (_@Configuration_).  
But those are loaded only if specific conditions are met - namely:

+ required class is available on the classpath (new beans magically created when dependency added)
+ required bean was not created explicitly by a programmer

So for example - to load _WebMvcAutoConfiguration_ when _Servlet_, _DispatcherServlet_, _WebMvcConfigurerAdapter_ classes are on the classpath _@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurerAdapter.class })_ is used.  
This configuration loads all web mvc defaults - but most of them again - only if specific bean doesn't yet exist.  
So for example - to check if _defaultViewResolver()_ should be created - _@ConditionalOnMissingBean(InternalResourceViewResolver.class)_ is used.


```java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
		WebMvcConfigurerAdapter.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
(...)
public class WebMvcAutoConfiguration {
	(...)

	@Bean
	@ConditionalOnMissingBean(InternalResourceViewResolver.class)
	public InternalResourceViewResolver defaultViewResolver() {
		InternalResourceViewResolver resolver = new InternalResourceViewResolver();
		resolver.setPrefix(this.prefix);
		resolver.setSuffix(this.suffix);
		return resolver;
	}
	
	(...)
}
```  

&nbsp;

#### Where Is It Triggered?

It all begins with _@SpringBootApplication_ - the one you annotate you main class with. If you check it's sources you'll find out _@EnableAutoConfiguration_ there - and this one is responsible for most of  the magic.

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Application.class);
        application.run(args);
    }
}
```  

If you check _spring-boot-autoconfigure.jar/META-INF/spring.factories_ you'll find out _org.springframework.boot.autoconfigure.EnableAutoConfiguration_ property which specifies which auto configurations will be used to 'guess' and create beans you require.

![autoconfigurations]({{ site.baseurl }}/images/2015-10-11-under-the-boot/autoconfigurations.png "autoconfigurations")

&nbsp;

#### DB Magic Example

So let's figure out how the required components for db access are created.  
To do so let's not use _'org.springframework.boot:spring-boot-starter-data-jpa'_ dependency in our showcase, but start with _'org.springframework.boot:spring-boot-starter'_ and see what dependencies should we add to create database aware app and how required steps are automagically performed.



With only _'spring-boot-starter:1.2.6.RELEASE'_ dependency I got 22 beans registered by spring in my app(18 spring beans + 4 application specific beans). There is no _dataSource_ nor _transactionManager_ amongst them.



I want to add hibernate entity to the project so I will include _'org.hibernate:hibernate-entitymanager'_ compile dependency.  
Now _JtaAutoConfiguration_ with _jta properties_ beans were added as _javax.transaction.Transaction_ appeared on the classpath.  
I was little hoping for _HibernateJpaAutoConfiguration_ to catch on - but this one also requires

+ _LocalContainerEntityManagerFactoryBean_ (to provide _entityManager_)
+ _EnableTransactionManagement_ (to provide _transactionManager_)

Both can be found in _'org.springframework:spring-orm'_ - so let's add this dependency to the classpath.  
Now spring will try to load _HibernateJpaAutoConfiguration_, but this one requires _dataSource_ bean (_@Autowired_ as a private field),
and we are still missing it.  
It's easy to figure out that _dataSource_ will be created by _DataSourceAutoConfiguration_ (found on list in _spring.factories_).  
It's seems that all conditions are met here, but _DataSourceAutoConfiguration_ class can't be classloaded yet - as it uses _org.apache.tomcat.jdbc.pool.DataSourceProxy_ so it depends on _'org.apache.tomcat:tomcat-jdbc'_ - let's add it to the class path as well.



Running the app we can see that we're getting closer - as this time _'hibernate.dialect'_ can't be determined. No surprise here - it couldn't have been determined by spring as we hadn't added any db specific dependency. So let's include '_com.h2database:h2'_ .



Everything seems to work now. 54 beans loaded by spring. Among them:

+ dataSource (_org.apache.tomcat.jdbc.pool.DataSource_ - added by _DataSourceAutoConfiguration.NonEmbeddedConfiguration.dataSource()_)
+ entityManagerFactory (_org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean_ - added by _JpaBaseConfiguration.entityManagerFactory()_ (parent of _HibernateJpaAutoConfiguration_))
+ transactionManager (_org.springframework.orm.jpa.JpaTransactionManager_ - added by _JpaBaseConfiguration.transactionManager()_ (parent of _HibernateJpaAutoConfiguration_)

&nbsp;

Dependencies are summarized on diagram below.

![dependencies]({{ site.baseurl }}/images/2015-10-11-under-the-boot/dbdeps.png "dependencies")

&nbsp;

Showcase project can be found ["here"](https://github.com/dkublik/under-the-boot).

&nbsp;
****



