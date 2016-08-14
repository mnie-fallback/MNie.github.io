---
layout: post
title: Azure Text Analytics Api - Are your users happy?
---

Hi!

My first post I would like to dedicate Azure, namely the function sentiment from Text Analytics api, from module Cognitive Services, of which the last I heard. 
The module itself gives us many opportunities, from text study to building recommendation systems. 
Api Text Analytics gives us the ability to detect sentiment, keywords, topics and the language in which text is written. 
In contrast, the above-mentioned function, examines the sentiment of the text. What does it mean? It can determine whether a statement/text has a positive or negative undertone. 
I decided it would be fun to "miracle" to test and see how it works.

So, where to start?
First of all, we need to have an account on Azure. Then we need to add to it a Cognitive Services module. 
Now it would be nice to throw our data (comments, posts, text, whatever we want to check) and generate some results.

But after that. First, we need to know what data format is accepted by the function. This format is json formatted as follows:

<script src="https://gist.github.com/MNie/a2a75632038748f9fc502b805f9706fd.js"></script>

In short, 'documents' is a list containing information about the data. Single object in the list of 'documents' contain 'id' which is a unique identifier. 
'Language', which is the language of a text from the variable 'text' in the case of the English language it is not required, due to the fact that it is the default language.

Okay, we know in what format data should be. So we go on. In my case, I have only comments that I have in the csv file, without any identifiers, so I'll have to somehow generate it. 
For this purpose I wrote a piece of code below:

<script src="https://gist.github.com/MNie/60c0ae7de329a09a342e780af11f3d56.js"></script>

Type 'requestJson' defines the input data, while the function 'createJson' is responsible for converting input data in .csv file to format required by the API. 
As we can see, it accepts a sequence of lines from a csv file, and then converts the sequence to the sequence of 'strings' in the last step it converts it to a sequence of type 'document'. 
By using the function 'mapi' I generated unique identifiers, super! I've also added the ability to save this JSON file (function saveJson). 
Comments in my case are in English, but I decided to arbitrarily set the attribute 'language' to 'en'.

We already have input, what now? Put it to the Azure! For this purpose I wrote a few more lines of code that sends the correct request to the Text Analytics API with properly formatted JSON. 
At this point we need a 'api_key', which can be found in the Cognitive Services module settings.

<script src="https://gist.github.com/MNie/85aebd24f5e1d14c3a7c744e661995f8.js"></script>

As we can see in the function 'getSentimentScore' it sends POST request, which in the header to the key 'OCP-APIM-Subscription-Key' assigned a 'api_key', while the body contains our JSON. 
Question is what will give us the api?
It will give us the data in the following format:

<script src="https://gist.github.com/MNie/29a7fd3bf84393304c53f3c176235db3.js"></script>

It is worth stressing here that the 'score' can vary from 0 to 1, where if the value is greater the text has a more positive undertone. As for the return, it looks okay. 
We using the 'id' to link the result to the text, cool. But I would like to see in one object id, scores and text to which it relates, for this purpose I wrote another piece of code.

<script src="https://gist.github.com/MNie/98e33c95f331bd3807adf646fec81d67.js"></script>

So, this is exactly what I meant! This function 'concatSentimentAndInput' first calls function that converts my comments from a csv file, the method required by the sentiment Text Analytics API, and then calls a function to query api, finally, to connect the input JSON together with the return from Azure. 
In addition, at the end results are sorted in ascending order, so at the beginning there are of the 'worst' comments.

That's probably all. As a result, I have a small little script that converts my comments, it does request to Azure for their semantic evaluation, and finally returns to me a sorted result combined with a set of an input.

Question function sentiment is working properly? Amazingly, the result looks as okay. 
Comments having a negative undertone have the lowest scores, while those positive indeed achieve a high rating, I did not notice any 'irregularities in the operation of the module.' 
Only puzzling is that the text 'good' has a higher rating from a 'very good' ... :)

![Image of good and very good](https://mnie.github.com/img/AzureTextAnalyticSemantic/good.png)

In summary, I would recommend anyone to try this function to determine how customers perceive your product, by examining their comments. 
In my case, the results came out really good and I'm going to use them in improving their applications.