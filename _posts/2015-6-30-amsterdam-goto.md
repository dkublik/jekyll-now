---
layout: post
title: GOTO Amsterdam 2015
comments: true
---

Amsterdam gave me a taste of everything - I guess this is what this city is about :).  
Organizers clearly  

+ knew how to please participants (city central location, very good catering, unlimited beers after speeches),
+ knew how to drag our attention (drones, lot's of t-shirts and other gadgets to give away),
+ knew how to organize schedule so everyone could find something for himself.

#### A little bit of this and a little bit of that

Staying away from java-script and docker paths I still tasted a lot of various things.  
A little academic knowledge at ["A Taste of Random Decision Forests on Apache Spark"](http://gotocon.com/amsterdam-2015/presentation/A%20Taste%20of%20Random%20Decision%20Forests%20on%20Apache%20Spark) by Sean Owen - Did not learn that much about Spark, but it was fun to warm up your brain with a little math, graphs, decision trees and formulas before other talks :).  
A little solution insight at ["Events storage and analysis with Riak at Booking.com"](http://gotocon.com/amsterdam-2015/presentation/Events%20storage%20and%20analysis%20with%20Riak%20at%20Booking.com) by Damien Krotkine and "The Evolution of Hadoop at Spotify - Through Failures and Pain" by Josh Baer and Rafał Wojdyła. More about these later.  
A further confirmation that microservice paths are still mostly about preaching and general introductions rather than details, solutions and specifics.  
Well - maybe not all of them. In ["Developing event-driven microservices with event sourcing and CQRS"](http://gotocon.com/amsterdam-2015/presentation/Developing%20Event-driven%20Microservices%20with%20Event%20Sourcing%20&%20CQRS) Chris Richardson did not limit himself to to give general descriptions (scalling, monoliths vs microservices, event sourcing and snapshots) but tackled also more specific issues that forces many microservice developers to spend sleepless night. Tech problems like _atomicity of event publishing and state updating_, _pros and cons of different event store implementations_ or _consequences of different architecture choices_ backed up by real life code examples - those were the things I would surely like to see more.

#### The need for data

Getting back to Booking.com and Spotify - two presentations, describing real life solutions, that especially got into my head.  
And I do remember both of them mostly because of the data volumes they mentioned.  
I know that logging and gathering the data is important, but was totally impressed by booking.com volumes: **15K events per s** and **100GB per h** of data to be stored.  
It's a lot to process and a lot to track. Damien reminded us of the importance of visualisation showing their graphs and dashboards.
Also showed that with that volume of data we need to rediscover things that we took for granted: json may not be efficient enough for communication, events may need to be aggregated before storing, data distribution encounters network capacity problems.

![Events storage and analysis with Riak at Booking.com]({{ site.baseurl }}/images/2015-06-30-amsterdam-goto/booking.jpg "Events storage and analysis with Riak at Booking.com")

&nbsp;

Still being under the impression of Booking.com presentation I was totally astonished with Spotify volume: **400TB of data generated per day!!!**  
So what can they do with that amount of data? Josh and Rafał were kind to explain.
For once - they can learn user's behaviour and adopt their services. It's obvious that Spotify gives song proposals based on your historic choices but also learns your behaviour - that you may prefer hard rock during morning jogging, jazz at a lunch time and classical music when laying in bed. They choose commercials so they are in the same mood like the music you are listening to and with newest feature they can propose your favourite songs matched to your speed rate when running. The power of data.

![The Evolution of Hadoop at Spotify - Through Failures and Pain]({{ site.baseurl }}/images/2015-06-30-amsterdam-goto/spotify.jpg "The Evolution of Hadoop at Spotify - Through Failures and Pain")

&nbsp;

#### Summary

Booking.com and Spotify gave away simple recipe for a good presentation: astonish with data, describe a problem, show an evolution of a real life solution. 
I hope to see more presentation like that in the future. Who knows - maybe we live to see a conference where general 'solution' track is replaced by 'real life production solution' topics. Hope to see it :).

&nbsp;
****



