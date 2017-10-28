---
layout: post
title: SpringBoot - Metrics with Servo and AWS CloudWatch
comments: true
---
Article explains how to send SpringBoot and Netflix Servo metrics to AWS CloudWatch.


#### TL;DR

If you are not interested how and why and just want to make it work do the following:

add dependencies (here build.gradle)

```groovy
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-aws')
    compile('org.springframework.cloud:spring-cloud-aws-actuator')
    compile('com.netflix.servo:servo-core')
    compile('org.aspectj:aspectjweaver')
}
```

set namespace name by property (application.properties)

```
cloud.aws.cloudwatch.namespace=m3trics

```

That's it. If you are interested why and how it works - please continue reading :).
You can also check and follow everything in [working code](https://github.com/dkublik/m3trics)

&nbsp;

#### SpringBoot Actuator

It's super easy to enable some metrics with [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html)

So let's create SpringBoot project with two dependencies:

```groovy
compile('org.springframework.boot:spring-boot-starter-web')
compile('org.springframework.boot:spring-boot-starter-actuator')
```

and in application.properties set

```
management.security.enabled=false
```

so that we are authorized to access all protected actuator endpoints (you may reconsider it in production)

Now launch the app and try: [http://localhost:8080/application/metrics](http://localhost:8080/application/metrics)

and we will see some metrics like:
mem, processors, heap, etc...

Check _org.springframework.boot.actuate.endpoint.PublicMetrics_ interface and it's class hierarchy to find providers for all presented metrics.

&nbsp;

#### CounterService & GaugeService

One of the _PublicMetrics_ providers is special - _MetricReaderPublicMetrics_ - because it reads all metrics registered by _CounterService_ and _GaugeService_.

What are they:

CounterService - registers tracks just inreasing a counter (so can be used to number of requests, pages visits, etc)

GaugeService - can store any value (it's not incremental - just set), so it's application may vary from from measuring time to showing info about threads, conntections, etc.

These services are used in many places.

For example - for every request to any endpoint at least two metrics are registered,
even for [http://localhost:8080/application/metrics](http://localhost:8080/application/metrics) we have:

```
gauge.response.application.metrics: 0.0
counter.status.200.application.metrics: 7
```

where 
_application.metrics_ is taken from url path -> _/application/metrics_

_counter.status.200_ means number of hits to this particular endpoint with response code = 200

_gauge.response_ - is last response time


The inner workings are simple. With _spring-boot-starter-actuator_  we added _MetricsFilter_ by _MetricFilterAutoConfiguration_ which basically is _javax.servlet.Filter_ submiting metrics by _CounterService_ and _CounterService_ for every request.


_CounterService_ and _GaugeService_ may also be used to create our own metrics, like in:


```java
@RestController
@RequestMapping("/favorites")
class FavoritesNumberController {

    private final GaugeService gaugeService;

    FavoritesNumberController(GaugeService gaugeService) {
        this.gaugeService = gaugeService;
    }

    @GetMapping
    void favoriteNumber(@RequestParam Double number) {
      gaugeService.submit("favoriteNumber", number);
    }
}
```


Now when we hit [http://localhost:8080/favorites?number=11](http://localhost:8080/favorites?number=11)

and we got:

```
gauge.response.favorites: 24.0,
counter.status.200.favorites: 3
```

submitted by _MetricsFilter_ but also our custom metric:
```
gauge.favoriteNumber: 11.0
```

_CounterService_ and _GaugeService_ are interfaces and by default they use in memory objects (_CounterBuffers_, _GaugeBuffers_) to store metrics.

To send it to the outside world _MetricExporters_ are used which in separate thread (_MetricExporters.ExportRunner_) will send the counter and gauge metrics wherever we want.
_MetricExportAutoConfiguration_ tells us that to make it work at least one _MetricWriter_ need to be present. None is registered but deafult but even with _spring-boot-starter-actuator_ we got components to write metrics as _MBeans_, send them to _StatsD_ (front-end proxy for the _Graphite/Carbon_ metrics server) or store them in _Redis_ (check different implementations of _org.springframework.boot.actuate.metrics.writer.MetricWriter_).

Nothing for CloudWatch here, but we can easilly get there.

&nbsp;

#### Sending Metrics to AWS CloudWatch


Now to be able to wire our metrics to CloudWatch we need _CloudWatchMetricWriter_ and we get it by adding

```
compile('org.springframework.cloud:spring-cloud-aws-actuator')
compile('org.springframework.cloud:spring-cloud-starter-aws')
```

to our dependencies.
In the second dependency we'll find _CloudWatchMetricAutoConfiguration_ which registers our _CloudWatchMetricWriter_.

If we check _CloudWatchMetricAutoConfiguration_

```
@Configuration
@Import(ContextCredentialsAutoConfiguration.class)
@EnableConfigurationProperties(CloudWatchMetricProperties.class)
@ConditionalOnProperty(prefix = "cloud.aws.cloudwatch", name = "namespace")
@ConditionalOnClass(name = {"com.amazonaws.services.cloudwatch.AmazonCloudWatchAsync",
        "org.springframework.cloud.aws.actuate.metrics.CloudWatchMetricWriter"})
public class CloudWatchMetricAutoConfiguration {
(...)
```

we will see that one more piece is missing: _cloud.aws.cloudwatch.namespace_
so let's set it to application.properties

```
cloud.aws.cloudwatch.namespace=m3trics
```


Now application can be deployed to the cloud, or run locally, if we have proper AWS credentials configured.
(For different types of providing aws credentials look at _DefaultAWSCredentialsProviderChain_ class).
More over, if you are running app locally also add property:

```
cloud.aws.region.static=us-east-1
```
(or other region you use) as Spring Cloud won't be able to figure it out if not running in EC2 instance.

Now access different endpoints in app and with few seconds delay you will be able to observe our counter / gauge metrics in AWS CloudWatch.
Just login to aws console, and to CloudWatch -> m3trics (remember the _cloud.aws.cloudwatch.namespace=m3trics property_)

![Registered Metrics namespace]({{ site.baseurl }}/images/2017-10-28-springboot-metrics/namespace.png "Registered Metrics namespace in CloudWatch")


![Spring Boot basic metrics in CloudWatch]({{ site.baseurl }}/images/2017-10-28-springboot-metrics/basic-metrics.png "Spring Boot basic metrics in CloudWatch")

&nbsp;

#### Netflix Servo Metrics


All of this is of course only half of the story.
With Spring Cloud we will use a lot of Netflix libraries, and these do not use Spring Cloud's _CounterService_ / _GaugeService_ but generate metrics on their own using Netflix _Servo_ and _Spectator_ collection libraries.
(you will find some info about these [here](http://cloud.spring.io/spring-cloud-static/Finchley.M2/#netflix-metrics), main point being - _Spectator_ is new and should replace old _Servo_ :))


Let's try with _Hystrix_ and add dependency our project:

```
compile('org.springframework.cloud:spring-cloud-starter-hystrix')
```

Now we got _spring-cloud-netfix-core_ library and inside we find _MetricsInterceptorConfiguration_, which is loaded conditionally on _servo_ package present, so let's add new dependency

```
compile('com.netflix.servo:servo-core')
```


We want to gather metrics from _RestTemplate_ as well and _MetricsInterceptorConfiguration_ specifies that this one is dependent on _aspectjweaver_, so let's add this depdendency as well.


```
compile('org.aspectj:aspectjweaver')
```

Now with _ServoMetricsAutoConfiguration_ we got _ServoMetricServices_ which is both _CounterService_ and _GaugeService_ implementation. So from now on, all gathered servo metrics will be sent to CloudWatch!


Let's check some _Hystrix_ in action.
Just enable it by _@EnableCircuitBreaker_, and create some hystrix traffic, like:

```java
@Service
class SomeTrafficGenerator {

    private final RestTemplate restTemplate;

    SomeTrafficGenerator(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Scheduled(fixedDelay = 5000)
    @HystrixCommand
    void makeSomeHit() {
        restTemplate.getForObject("https://www.google.pl", String.class);
    }

}
```

Run the application one more time and enjoy plenty new metrics in CloudWatch!

![Hystrix Metrics namespace in CloudWatch]({{ site.baseurl }}/images/2017-10-28-springboot-metrics/hystrix-metrics.png "Hystrix Metrics namespace in CloudWatch")


&nbsp;

#### Spring Boot and Spectator metrics

Now you might be tempted to do the same with _Spectator_ by adding
```
compile('org.springframework.cloud:spring-cloud-starter-netflix-spectator')
```

but I don't think Spring Boot is ready to work with it.

Add spectator as dependency and _ServoMetricsAutoConfiguration_ will no longer be loaded (_@ConditionalOnMissingClass("com.netflix.spectator.api.Registry")_),
so no _ServoMetricReader_ will be registered, so _MetricsExporter_ will no work.
(there is _SpectatorMetricReader_ in _spring-cloud-netflix-spectator_ library but for some reason it's not registered by _SpectatorMetricsAutoConfiguration_).

&nbsp;

In the time of wriritng the newest Spring Cloud library is Finchley.M2 (and Finchley is only version working with Spring Boot 2.0) - so maybe again eveything will work with GA version.

&nbsp;

Again you may check the code [here](https://github.com/dkublik/m3trics)

&nbsp;

