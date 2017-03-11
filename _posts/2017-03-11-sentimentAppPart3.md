---
layout: post
title: Creating a fully functional F# microservice (Quartz.Net, Net.Mail)
---

Welcome back,

In previous posts we learned how to create a project in Nancy F#. How to configure Fsharp.Configuration, SqlTypeProvider and how to use the Azure Api A. As you probably remember mikroservie had in his intention to send periodically e-mails with information about what new evaluations appeared and how their sentiment looks like.
In order to achieve this purpose I decided to use the Quartz.Net library for job scheduling and System.Net.Mail to send e-mail messages.
I'll start with the configuration of the Quartz.Net, starting with adding following record to paket.dependecies file:
<script src="https://gist.github.com/MNie/be440835382de01b518f91503de1437e.js"></script>
We create a new file in which we place the code of our task. We start by creating a class that will inherit IJob interface. This class should implement the Execute method and perform the action that should be performed periodically.
In our case, this method will look like this:
<script src="https://gist.github.com/MNie/a32431e0d41254f3852e1570b70989e2.js"></script>
As we can see, first we get data, then we ask azure about sentiment and then send e-mails(details about sending emails in a moment).
Then I decided to create a new module a job factory. So by we could easily schedule a job in main.fs and keep (main.fs) free of Quartz namespaces. The function looks like this:
<script src="https://gist.github.com/MNie/f55cbe296a87848421307f997cef7e1e.js"></script>
As we can see in the method schedulejob we create an SchedulerFactory object, create a job of type previously created by us type("Job") and define starts periodicity. Where periodicity is defined by the following method:
<script src="https://gist.github.com/MNie/a2117b8a710be61b9efc48a7d17ae685.js"></script>

As you can see periodicity is set that our action is invoked every week.
Well, we have created Job class and its factory, we should call somewhere scheduleJob method from a fabric. So we add method invocation to a main.fs file:
<script src="https://gist.github.com/MNie/9cc41dc64d1f5bb6eaf7a75d7044251a.js"></script>
And its all. At this point we have done task scheduling. The final aspect worth explaining is mail sending that left for the end.
To send mail I used a System.Net.Mail library. I started by creating a new file and place there a new class(MailSender). This class contains a single method that is available outside, a send method.
It is designed based on input data to build a body of the message and then send it from/to email defined in a configuration file.
<script src="https://gist.github.com/MNie/4ead7df10bc6febd6bb9e195d2394aca.js"></script>
Sending method looks like this:
<script src="https://gist.github.com/MNie/c2117667704845dbd661e41512499eb7.js"></script>
Method for creating a message like this:
<script src="https://gist.github.com/MNie/384196478c08e0265ee35dfa196b6c95.js"></script>
An interesting fact here is setting observer Who has various "tasks" to be performed when an error occurs (for example printing an error msg to console and above all call a Dispose method)
Of course, we will add new endpoint (method in nancy module) which allows sending e-mails. It looks in the following way:
<script src="https://gist.github.com/MNie/a1275e34534c41ecd3d91f95b366ae4d.js"></script>
![result](https://mnie.github.com/img/11-03-2017SentimentAppPart3/result.png)
Summarize all posts we can see how easily we could create a small microservice that contains a couple of functionalities. Comparing it to do similar/related service what I created in C# I could say that purity of code due to its greater brevity is way better. We also eliminate a lot of redundant lines of code (boilerplate code) and the complexity in code, which is often imposed by the selected technologies (dependency injection configuration, database configuration, etc.). In my opinion this example very nicely shows how we can "facilitate" Creating a microservices by using F#, in comparison to making them in C#.

Thanks for reading!
[Source](https://github.com/MNie/SentimentNotifier)
[Previous post](https://mnie.github.io/2017-03-11-sentimentAppPart2/)
