---
layout: post
title: Spring I/O 2015 Barcelona
comments: true
---

Few days ago 4F company gave me an opportunity to participate in Spring I/O conference in Barcelona.  
2 days, 4 rooms and 41 speakers presenting technologies behind, within and around most popular java framework.  
Having some conference history, at some point I developed believe that at any given event, many presentations and lectures find their place in agenda only by mere accident. All the more I was curious how all of this will look under watchful (I hoped) eye of Pivotal.

#### First impression

Event appeared to be very cameral.  
With just a little more than 300 participants one wasn't lost within enormous crowd between lectures or crushed in never ending line for a lunch.
In the same manner there were only few stands from sponsors. Not that many locals as well - crowd seemed to be rather international.
Lectures in small rooms - easy to ask questions, easy to focus.

#### Lectures

With spring centric approach many lectures, in my opinion, followed similar pattern - showing how easy it is to start project with spring boot and presenting given lecture topic just as the minor part of described solution.  
That was for example the case with [MongoDB presentation](http://www.springio.net/mongodb-and-spring-two-leaves-of-a-same-tree/).
When I was expecting to see some Mongo magic all I got was another tutorial for setting spring boot app from the scratch.
Don't get me wrong - it's all cool and everything but everyone knows it for some time now, and many presenters acted liked they were revealing some mysteries to us - when in fact - almost no one was.
I was little disappointed by that - cause seeing some magic was what I was secretly hoping for.
I grow little tired with mere basics on other conferences - and hoped - that under wings of Pivotal - I'll finally see how things are handled in leading projects and leading companies, hear about their problems, solutions and real live experience.
Instead - in most cases - I got what can be found in basic tutorials.  
There were some pearls of course - ["Inside an Spring Event Sourced CQRS application – or why Microservices can actually work"](http://www.springio.net/inside-an-spring-event-sourced-cqrs-application-or-why-microservices-can-actually-work/) - showing step by step evolution of monolithic application to inevitable end as (almost) microservice solution or ["Everything you need to know about Java Classloaders"](http://www.springio.net/everything-you-need-to-know-about-java-classloaders/) where Oleg Šelajev presented exception by exception what can go wrong behind jvm classloaders curtains.
But the presentation I liked most was

#### ["Performance Testing Crash Course"](http://www.springio.net/performance-testing-crash-course/) by Dustin Whittle.

Which I found very surprising as I hate performance.
However - topic was presented brilliantly. Dustin talked fast and concretely, covering topic from cover to cover.
Real live problems, real live experience. No time loosing here.  
Starting from business point of view, showing larger context, technical loss and gains, Dustin presented performance as the first class project feature.  
Place your app _("cf push")_, understand performance on different levels _("Static vs Hello World vs Application")_, then attack it.
At first gently (with linux _ab_ and _siege_ command line tools), discover all points to attack _(sproxy)_, then, apply more sofiscicated scenarios _(jmeter, multi-mechanize)_.
Again - see the bigger context - where you have to attack from more than one node. Call the bees with machine guns _(pip install beeswithmachineguns)_ for help.
Do not stop on server - test client side as well _(Google PageSpeed, WBench, Grunt)_. Monitor statistics _(Statsd + Graphite + Grafana)_, track performance of end users _(boomerang, webpagetest.org, SiteSpeed.io, appdynamics)_.
Dustin shared recipes for treating it all.  
That knowledge was really worth acquiring. I wonder how many hours I would spend googling it all when I received it on my plate. This is what differed this presentation from many others merely repeating [https://spring.io](https://spring.io) guides.

![Performance Testing Crash Course]({{ site.baseurl }}/images/2015-04-11-springio/perf_1.jpg "Performance Testing Crash Course")

&nbsp;

#### Summary
Spring I/O - I liked it. How could one not. With everything above said it was worth seeing. I do realize that I'm one of the luckies. My company aims for new technologies and is open for new ideas. We are always in chase for changes, but I do know that not everyone is. For some - presented things could open new perspectives. For those of us already familiar with it - it's still good to know that you are on the right track.

![Lunch break]({{ site.baseurl }}/images/2015-04-11-springio/hotel_1.jpg "Lunch break")

&nbsp;
****



