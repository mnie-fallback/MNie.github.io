---
layout: post
title: Creating a fully functional F# microservice (NancyFx, SqlTypeProvider)
---

Hi!
 
recently I realized that I have the data where users evaluate things presented to them. These opinions are stored in the database, but let's be honest how often you open a management studio to see what new records appear in the database? I am definitely not that kind of person. So I thought it would be fun to write an application that will read new data from the database, send them to azure to investigate their "sentiment" (and in some way help me in their analysis), then combined them with an input data and sent as a weekly report and also providing an api, to ask for data from the last week.
Having experience from daily work with Quartz.net and Nancyfx I thought it would be fun to write an application using precisely these technologies. In addition, I want to use SqlTypeProvider and, of course F#.
Due to the use of many different libraries, I divide this post to smaller ones.

In this post I want to focus on getting data from the database and make them available as json via endpoint for a user.
We start by creating a NancyFx project f#, press ctrl + shift + p in visual studio code and select:
![new project](https://mnie.github.com/img/11-03-2017SentimentAppPart1/newproject.png)
![new nancy](https://mnie.github.com/img/11-03-2017SentimentAppPart1/newnancy.png)
Right now we have empty NancyFx project (we have to remember about running Visual Studio Code as an administrator, since NancyFx have to register a port when running).
So let's start by extracting data from the database. As I mentioned earlier, we use SqlTypeProvider. We start by adding references to packet.dependencies:
<script src="https://gist.github.com/MNie/3b0e2e276bf1ebf30c6aeee132889db0.js"></script>
As you can see we pass an configuration file name and connection string key instead of passing a whole connection string as a parameter.
Let's start by creating a new file, which will be responsible for downloading data from the database. The structure of the table in which opinions of users are stored looks as follows:
![db](https://mnie.github.com/img/11-03-2017SentimentAppPart1/db.png)
Let's start mapping it in code:
<script src="https://gist.github.com/MNie/df9e0df19870cc3416fdf24f27696d01.js"></script>
Well, we have a ready structure. We could move on to getting data from the db. We want to extract data just before last week. So we begin with setting schema file:
<script src="https://gist.github.com/MNie/5c9b24c663eaf85f325f162ca9135dc2.js"></script>
<script src="https://gist.github.com/MNie/951a6c09c3bd995ba30f731206f563ed.js"></script>
Pull out the context and set the question
<script src="https://gist.github.com/MNie/5eac0365760da845b03e2761d3dcbddd.js"></script>
As we can see query is similar to SQL, so it is self explanatory.
In the last step we turn mapped to record from the database to a predefined type.

That's it? So these few lines enough to drag data from a database. Since we want to share this data also in the endpoint, let's add an action to the nancy module nancy, let's call it "RawData"
<script src="https://gist.github.com/MNie/f477e875a279f10c79eeeada5cd5c148.js"></script>
![raw data](https://mnie.github.com/img/11-03-2017SentimentAppPart1/RawData.png)
Well it would be enough when it comes to this post in the next post, I want to take care of adding support for Azure api to examine sentiment messages.
 
To sum up, in just ~10 lines of code written, service gets connection string from a configuration file, connects to the database and pulls data from it. It looks very neat, and above all straight. In my opinion much better than the analogous code written in C#.

Thank you for reading!


* [Source](https://github.com/MNie/SentimentNotifier)
* [Next post](https://mnie.github.io/2017-03-11-sentimentAppPart2/)


