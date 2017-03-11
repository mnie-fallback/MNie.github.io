---
layout: post
title: Creating a fully functional F# microservice (Azure, FSharp.Configuration)
---

Welcome back,

in the [previous post](https://mnie.github.io/2017-03-11-sentimentAppPart1/) I described the setting of Nancy and SqlTypeProvider in the project. Whole application was designed to periodically send information with incoming customer evaluations about products presented to them as well as evaluating their (evaluations) sentiment.
At this time we have finished downloading data.
In this post I would like to take care of adding the connection with an Azure Api to download semantic evaluation of the aforementioned comments.
For this purpose we use sentiment function from the Text Analysis Api module from Azure Cognitive Services (more about the function and how it works [here](https://mnie.github.io/2016-08-14-TextAnalysis-SemanticFunction/)). In short, the function based on submitted text(s) returns information How undertone had given text. Whether it was positive or negative in 0-1 scale where the higher the value is, the more positive undertone it has. We begin by preparing format for Azure Api, So I created this types:
For request (to Azure):
<script src="https://gist.github.com/MNie/2b544acd527baa66c3811ddb7a504b1b.js"></script>
For response (from Azure):
<script src="https://gist.github.com/MNie/8030f0755b84ad734592f58c1dd93597.js"></script>
As we remember, the data model from a database is a little more complicated
<script src="https://gist.github.com/MNie/df9e0df19870cc3416fdf24f27696d01.js"></script>
We have to map somehow data from to db to fit into data accepted by Azure api. So I created following function to map data:
<script src="https://gist.github.com/MNie/282a8d9357dadc8240f721b65f66d4f0.js"></script>
Well we have a ready data conversion "to" Azure Api. So now we need to call the action "sentiment". Call looks as follows:
<script src="https://gist.github.com/MNie/5e9487b2bc716b24489cbdfe31582b30.js"></script>
At this point we should have a result from Azure and the data from db. The only thing left for us at the moment is to map these data together, I realize this with this method:
<script src="https://gist.github.com/MNie/cbb826fe7441ec106810379be9f1b262.js"></script>
To check if everything works nicely, I provide a new action in the nancy module:
<script src="https://gist.github.com/MNie/28b3525323c16026cc9db0f3c3b1ddd4.js"></script>
![Azure](https://mnie.github.com/img/11-03-2017SentimentAppPart2/estimate.png)
Everything looks great, the only thing which bother me are hardcoded strings in code, they are bad.
It is difficult then to manage such code, which contains somewhere some magical values. Let us, therefore move these values to the configuration file (app.config) so we could read them later from it in code. To achieve that I add reference to FSharp.Configuration:
<script src="https://gist.github.com/MNie/858a33f0bbc1c6f5808a9a72144d901a.js"></script>
Well, but we need to extract data from the configuration file? Are we use CofnigurationManager.AppSettings for that? Well, no, we don't have to install FSharp.Configuration if we want to use ConfigurationManager :)
Let's use this library to get our configuration. So we define a Settings type as follows:
<script src="https://gist.github.com/MNie/0d419408b141e23f01c85a800ebb882b.js"></script>
So it could read configuration from an app.config file.
<script src="https://gist.github.com/MNie/18a35872667daa0d9afc20b756c9fba4.js"></script>
How we can get values from an app settings? Simple as that:
![Settings](https://mnie.github.com/img/11-03-2017SentimentAppPart2/Settings.png)
As we can see all values from and app.config are available here and what is very cool it understand types in app.config. So AzureApiUrl has type Uri not a string. So function to get data from Azure looks right now like this:
<script src="https://gist.github.com/MNie/43d1808ea4f6421b8e383c2293159507.js"></script>

So it's all for this post. We saw ho to pair calling external API in our mikroservice and how to easily extract the data we need, from a configuration file by using FSharp.Configuration. What has to be done to finish the project is a cyclical sending of messages, what I would like to take care of in next post.

Thanks for reading!
[Source](https://github.com/MNie/SentimentNotifier)
[Previous post](https://mnie.github.io/2017-03-11-sentimentAppPart1/)
[Next post](https://mnie.github.io/2017-03-11-sentimentAppPart3/)
