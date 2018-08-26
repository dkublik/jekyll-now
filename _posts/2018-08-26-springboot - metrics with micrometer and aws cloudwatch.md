---
layout: post
title: SpringBoot - Metrics with Micrometer and AWS CloudWatch
comments: true
---
Some time ago I wrote a blog about how to configure CloudWatch metrics Spring Boot. It was all before micrometer and depended on Netflix Servo Metrics.
Time was slowly passing and civilizations prospered but it's still difficult to find info on how to make Spring Boot work with Micrometer CloudWatch.
So here it goes.


#### TL;DR

If you are not interested in why and just want to make Spring Boot work with AWS CloudWatch do the following:

* add dependencies ([build.gradle](https://github.com/dkublik/micrometer-aws-example/blob/master/build.gradle))

```groovy
dependencies {
	compile('org.springframework.boot:spring-boot-starter-web')
    compile('org.springframework.boot:spring-boot-starter-actuator')
    compile('org.springframework.cloud:spring-cloud-starter-aws')
    compile('io.micrometer:micrometer-registry-cloudwatch:1.0.6')
}
```

* set required [properties](https://github.com/dkublik/micrometer-aws-example/blob/master/src/main/resources/application.properties)

```
management.metrics.export.cloudwatch.namespace=m3trics2
management.metrics.export.cloudwatch.batchSize=20
```

That's it unless you are interested in details - then please continue reading :).
You can also check and follow everything in the [working code](https://github.com/dkublik/micrometer-aws-example)

&nbsp;


#### Micrometer.io and CloudWath Metrics Exporter


It's not my goal here to describe Mirometer itfself, neither the concept of different metrics - as all the info you can be easilly find in [micrometer docs](https://micrometer.io/docs).

Setting up Micrometer with Spring Boot is super easy and it's mostly just adding a specified registry as a dependency, where the registries are different metrics systems that again are listed on [micrometer page](https://micrometer.io/docs).

Study the list however and you will not find CloudWatch there. Still, the registry exists and can be found both in [repo](https://repo.spring.io/libs-release/io/micrometer/) and 
[github](https://github.com/micrometer-metrics/micrometer/tree/master/implementations) waiting to be used.

When making particualar metric system ((ike e.g. datadog) work with Spring boot through micrometer all required componets can be found in two places:

* micrometer-registry-datadog.jar (in case of datadog) - containing Spring Boot independent meter registry and utils

* spring-boot-actuator-autoconfigure.jar - where we can find autoConfigurations for different systems (like [DatadogMetricsExportAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-actuator-autoconfigure/src/main/java/org/springframework/boot/actuate/autoconfigure/metrics/export/datadog/DatadogMetricsExportAutoConfiguration.java)) - automatically creating exporter for metrics system when given registry is present on the classpath.


Situation with CloudWatch however is a little different and we won't find AutoConfiguration for CloudWatch in actuator jar.
The thing is - when creating metrics exporter for CloudWath we will need Amazon CloudWatch client. With region providers, detecting if app is running in the cloud or not, login profiles, etc - such a client wouldn't be that trivial piece to write. Luckilly everything is already written - but not in Spring Boot but in Spring Cloud.
Because it's Spring Cloud that should depend on Spring Boot, not the other way around - _CloudWatchExportAutoConfiguration_ can't be put together with other systems in Spring Boot's actuator. So to use it we need to add one more dependency:

* spring-cloud-aws-autoconfigure.jar
(spring-boot-actuator-autoconfigure.jar is still needed because of classes like StepRegistryProperties)


#### Properties

Check the [CloudWatchExportAutoConfiguration](https://github.com/spring-cloud/spring-cloud-aws/blob/master/spring-cloud-aws-autoconfigure/src/main/java/org/springframework/cloud/aws/autoconfigure/metrics/CloudWatchExportAutoConfiguration.java) and you will find that the only property needed to enable metrics export to CloudWatch is

```
management.metrics.export.cloudwatch.namespace=m3trics2
```

As name suggests - property names the custom CloudWatch namespace where you will find your metrics.

![Registered Metrics namespace]({{ site.baseurl }}/images/2018-08-26-springboot-metrics2/cw-custom-namespace.png "Registered Metrics namespace in CloudWatch")
Registered Metrics namespace in CloudWatch

&nbsp;


Because of the bug in Cloud / Micrometer code one more property is needed

```
management.metrics.export.cloudwatch.batchSize=20
```

To undersand it you need to know that metrics are sent to CloudWatch asynchronously in batches and Amazon CloudWatch client has a limit of max _20_ metrics sent in single batch.
[io.micrometer.cloudwatch.CloudWatchConfig](https://github.com/micrometer-metrics/micrometer/blob/master/implementations/micrometer-registry-cloudwatch/src/main/java/io/micrometer/cloudwatch/CloudWatchConfig.java  handles it corretly),
but then actual properties are taken from [org.springframework.cloud.aws.autoconfigure.metric.CloudWatchProperties](https://github.com/spring-cloud/spring-cloud-aws/blob/master/spring-cloud-aws-autoconfigure/src/main/java/org/springframework/cloud/aws/autoconfigure/metrics/CloudWatchProperties.java) and this one just takes the _batchSize_ from property or sets default value to _10000_ not caring about AWS restriction.


#### Example metrics

Example code generating metrics can be found [here](https://github.com/dkublik/micrometer-aws-example).
It's a super simple web application generating random number every time a particular url is hit (/lucky-number). There is also scheduler hiting this url every 5 seconds to generate demo traffic.

App demonstrates that dozens of metrics are added automatically by Spring Boot - like:
* http.server.requests.avg - how long it took on average in one minute to respond with lucky number,

![90th percentile for endpoint reponse time]({{ site.baseurl }}/images/2018-08-26-springboot-metrics2/metric-avg.png "90th percentile for endpoint reponse time")
90th percentile for endpoint reponse time

&nbsp;

* http.server.requests.count - how many request where made in one minute

![Number of requests in one minute]({{ site.baseurl }}/images/2018-08-26-springboot-metrics2/metric-count.png.png "Number of requests in one minute")
Number of requests in one minute

&nbsp;

I also added one custom metric - luckies.current (check [LuckyNumbersController](https://github.com/dkublik/micrometer-aws-example/blob/master/src/main/java/pl/dk/m3trics2/LuckyNumbersController.java)) which just stores last lucky number that was generated.
(keep in mind that not all generated numbers will be send to CloudWatch - as gauge metrics used here is probed peridically - details can again be found in [micrometer docs](https://micrometer.io/docs))

![Last generated lucky number]({{ site.baseurl }}/images/2018-08-26-springboot-metrics2/metric-current-lucky.png "Last generated lucky number")
Last generated lucky number

&nbsp;

Again you may check the code [here](https://github.com/dkublik/m3trics)

&nbsp;

