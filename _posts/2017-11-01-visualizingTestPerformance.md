---
layout: post
title: Visualizing performance tests execution times 
---

Hey,


The post is actually a continuation to the previous article about performance testing in F#, which you can find [here](https://mnie.github.io/2017-08-17-performanceTestsUsingFsCheck/). 
Briefly, the clue of the previous post was to describe a solution based on the FsCheck library that tested the APIs available in the application, in terms of execution time for random http requests. 
Each test, depending on its type, had a predetermined minimum execution time that should be met.

How to determine the minimum execution time(s) for single test? 
Is the time for generating a table with one million records and two columns in less than a second at the beginning of a project could be somehow reliable relative to the time it generates "the same" table containing one million records with around 100 columns at the end of the work associated with it?


The question is, whether these times should always be the same throughout the entire development process, rather not, because what if you add more functionality, which in some way will cause the slowdown of application, or move application to the cloud, or upgrade/downgrade the machines on which it runs? 
For this purpose, I decided to monitor the test execution times in time. How to approach this? Times can be collected in the form of logs and then processed by a tool dedicated to it like a kibana. This task can also be accomplished using, for example, FsCharting or highcharts. 
However, due to the use of Kibana in my project. I decided that it would be a good idea to save each of the test times to ElasticSearch so that the results could then be easily available through the Kibana Dashboards.


We start by downloading and starting [elasticsearch](https://www.elastic.co/downloads/elasticsearch) locally. 
We launch it with the elasticSearch command in the bin folder in the previously downloaded source. ElasticSearch should be available by default at port 9200, which we can check by typing this address (localhost: 9200) in the search bar of a browser.
In addition to ElasticSearch we will also need [Kibana](https://www.elastic.co/downloads/kibana).


We run it like elastic, launching kibana.bat in the bin folder.


At this point we have an entire environment ready for saving test times. So we can go to the data which we would want to save in ElasticSearch.

This was accomplished by adding some functionality to the test project.
We start by adding the [NEST](https://github.com/elastic/elasticsearch-net) library to the project.


Going back to the previous post, the structure of the test looks as follows.

<script src="https://gist.github.com/MNie/ce25b2879ea2c421deed17f978baabd1.js"></script>

You may notice that sending information to ElasticSearch with the execution time should occur just before the assertion phase.
Due to the fact that sending this information will take place in many tests I decided to create a new module that will only answer for sending information to ElasticSearch.
This module looks like this:

<script src="https://gist.github.com/MNie/1dbdc59c9aae5deb52e6ef642eeb52fc.js"></script>

At the beginning you could se declaration of a model which will be represented test execution information.
Then there are 3 private functions that will help us perform operations on methods available in the NEST library.


The first one.

<script src="https://gist.github.com/MNie/c8d6dff29654e204a1cbfb0e39f76439.js"></script>

Is responsible for translating the function F# into a Func delegate from C#, the next one:

<script src="https://gist.github.com/MNie/fd566feab0332b9ae4a770ee1158503a.js"></script>

Is responsible for setting the IndexName based on a string argument. The last:

<script src="https://gist.github.com/MNie/d48465a099b124635e287c0b776ebcb5.js"></script>

Performs an index function with the specified argument and maps its result to the IndexRequest.


By going to the ElasticSearch connection setup, we can see the creation of an URI where we say where our ElasticSearch instance is located. 
Then we create a connection setup and create an ElastcSearch client and set up an index where our data will be available. 
An important aspect here is that the index name must be composed of only lowercase letters.


By going to the code that executes the action, you can see the senddata method, which creates a object that will represent the data held in ElasticSearch. 
Then in the try/catch block we see the message sent to ElasticSearch.


So finally structure of a test would look like this:

<script src="https://gist.github.com/MNie/dc4f7d784ba4ada032cd888bc4274ba1.js"></script>

It would be all about the code, after firing tests and letting them run several times, you will notice that we can create some visualizations that will show us how time executions looks like.
Which would look like this:

![elastic](https://mnie.github.com/img/1-11-2017VisualizingPerformance/2.png)
![kibana](https://mnie.github.com/img/1-11-2017VisualizingPerformance/1.png)

In conclusion, we can graphically see how long our test runs over time, and if we should reconsider changing time limits previously set for tests.

Thank you for reading :)
