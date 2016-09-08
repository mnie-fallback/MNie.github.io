---
layout: post
title: Azure functions in F#
---

Hello,

Today's post I would like to dedicate 'Azure functions' using the F#, which platform is recently released. 
Because I wanted to try it out and see what opportunities it gives us and how its creation and modification looks like.

In order to test the function I decided that I would write a simple application that will have to convert an array of numbers, ​​passed as an argument, to an array containing the new values.
But before we get to move on to implementation. We have to setup 'Azure functions'. There are 2 approaches to create azure's function. The first approach is based on "clicking" everything on Azure Portal, 
while the second one is based on an azurefunctions packet in npm and integration with GitHub or bitbucket.
First of all, we need to start by creating 'Azure functions' application, in which we can create azure functions.

Okay, we go to the 'Azure functions' application creation.
We begin by going to the module ['Azure functions'](https://azure.microsoft.com/en-us/services/functions/).
After log in to your account, You should see following page:

![Log in page](https://mnie.github.com/img/AzureFunctions/getStarted.png)

In the combobox on the left, we select our subscription. Combobox in the upper right corner is used to select existing azure functions. 
While Textbox and combobox at the bottom right allow us to create a new application and also select a region in which it should be. 
Let's start by creating a completely new 'application'. Name it "fancyFunction."

![app creation](https://mnie.github.com/img/AzureFunctions/appCreation.png)

Azure will create our application, and you will see a screen with the possibility to select a language in which function would be written and the 'trigger' type of our function for example QueueTrigger etc.

So we are at a point where we can begin with creation of our azure function!
As I said before, there are two approaches to create the function.
Let's start with the first one!

__Creating Azure Functions via Azure portal__

As we can see we have no choice of F#, the only choice is C# or JS. Something here is probably wrong, we should be able to create a function in F#!?

![init screen of function creation](https://mnie.github.com/img/AzureFunctions/functionCreation.png)

The whole thing is that, we should click the inscription at the bottom of the page 'Or create your own ..', so we do that.
A screen where we can choose the available templates should appears, we are interested in F# and an QueueTrigger as a function trigger type.

![advanced creation](https://mnie.github.com/img/AzureFunctions/functionCreationADV.png)

When you click on QueueTrigger we can name the function, select the queue (due to the fact that we chose it as a trigger), for which function will react and a connection string to the database.
Okay we created an empty function. The first thing I would like to change in it, is that the function should be triggered by a http request instead of a message in a queue. 
For this purpose we need to modify the JSON file, which is responsible for the 'triggering' of our function. 
Question where can we find it? To do this, go to the tab Integrate. Then click on 'edit advance', following json should appears:

<script src="https://gist.github.com/MNie/e7b8fbf4847c48e7f34cf1ff1a5d94c2.js"></script>

Let's change it as follows:

<script src="https://gist.github.com/MNie/38af80a1eac83229fcdad623a66f52e0.js"></script>

At this point, our function will be set to 2 bindings, one input for a http requests and one output. 
It is worth noting here that we set the level of authentication for anonymous, so anyone can call our function. 
We can then change that so there would be a need to transfer an api_key to a function, so that people with no priveledges would have no ability to call our function(activation of this feature is described in detail in the standard view tab Integrate).

Let's return to the tab Develop. After changing the trigger type of our function (change JSON). 
We should note that the place where we can write code, there is a section with a link to our function:

![code](https://mnie.github.com/img/AzureFunctions/code.png)

It will be needed later.

At this point I would like to move to the second way of creation of the azure function, so using the package azurefunctions available by npm.

__Azure funtions from the azurefunctions package in npm__

So where to start? Start the console in a place where You will have sources of a function and type the following command:

<script src="https://gist.github.com/MNie/299f79411554b6e3b5d74b9895c20467.js"></script>
![npm](https://mnie.github.com/img/AzureFunctions/npm.png)

Install globally module azurefunctions, which will allow for the integration of the Azure module azurefunctions. 
The next command you need to run is:

<script src="https://gist.github.com/MNie/d61cb78d1e6ae49f88288730f2feb136.js"></script>
![npm](https://mnie.github.com/img/AzureFunctions/init.png)

It creates an empty repository, along with the original files .gitignore, host.json and .secrets. 
We are also informed that we can create our new first function with the command:

<script src="https://gist.github.com/MNie/bf3c7492b75d97c54f66519220d26620.js"></script>
![npm](https://mnie.github.com/img/AzureFunctions/new.png)

This command will launch our new console wizard of function creation. We will have to choose in which language we want to create a function, type of trigger and name of a function.
At this point it is worth to turn our configuration file (located in the following location: {functionName}/function.json) of triggers, so that the function will work with http requests. 
To do this, open the file {functionName} /function.json and change it in a similar way as in the previous case. So we change its contents from:

<script src="https://gist.github.com/MNie/e7b8fbf4847c48e7f34cf1ff1a5d94c2.js"></script>

to

<script src="https://gist.github.com/MNie/38af80a1eac83229fcdad623a66f52e0.js"></script>

Okay, at this time our function is configured to run in the event of a http request. 
At this point, we should somewhere check our function (github, bitbucket etc.). 
After this 'operation', there is a time to integrate our function from Azure!

Time for Continuous Integration!
For this purpose, we enter the [Azure Portal](https://azure.microsoft.com/en-us/services/functions/) choose previously created application 'azurefunctions' and go to the Settings panel:

![CI](https://mnie.github.com/img/AzureFunctions/CI.png)

Click on the Continuous Integration:

![CI](https://mnie.github.com/img/AzureFunctions/CI2.png)

Click Setup, and then 'Configure required settings' and choose our code repository, in my case it's github.

![CI](https://mnie.github.com/img/AzureFunctions/CI3.png)

After filling in all the fields that appear when you select the type of code repository and a successful combination in the Deployments window we will be able to see commits that 'came' to Azure.

![CI](https://mnie.github.com/img/AzureFunctions/CI4.png)

In addition to the CI, we can still configure preformance tests of our function. 
We have the opportunity for them to define the time during which request would be generated and the number of 'users' who are going to generate these requests.

![CI](https://mnie.github.com/img/AzureFunctions/CI5.png)

Okay, at this point we are on a par with the creation of a function through the portal azure and module azurefunctions in npm. 
Next I will discuss everything on the example of the module azurefunctions in npm, due to the fact that on azure portal everything should be similar. 
So we go eventually to code .. hooray!
As we have seen already, Azure generates the "basic" version of our script file run.fsx, which looks like this:

<script src="https://gist.github.com/MNie/998b9648a3486c5884927294498fa451.js"></script>

As you can easily guess, the starting point of our function is the 'Run' method and that it is running at the moment of function call. 
It is not yet prepared to accept http requests and return responses (input argument to the function is a string, and in our case HttpRequestMessage is required). 
Because of that we have to convert a code to function like this:

<script src="https://gist.github.com/MNie/34b3c6e7929a2b1e9ec5b6530bb2a22b.js"></script>

The first thing that catches the eye is a change of the input argument of Run method for HttpRequestMessage. 
This will allow us to call function using the http request. 
In the next step we get all arguments from a request and we searched for an argument named: 'data' ignoring the case sensitivity, then we return its value and create a response. 
The answer is in JSON format(which tells us ContentType of a response). 
We have right now a basic version of a script which will be triggered by a http request and it should return a string so we are now ready to add some business logic to it. 
As I said at the beginning of the post, I wanted to write a simple function that will generate for me a new identifiers for input identifiers transmitted in the form of serialized array. 
In order to achieve this goal I add part of the 'business logic' to the script, which now looks like this:

<script src="https://gist.github.com/MNie/23af0ee11bcd7b91171209d1aa83d66f.js"></script>

As you can see I had to add a new namespace for Newtonsoft.Jsona added a new method GenerateIds, 
which is responsible for the creation of new ids. 
For the sake of explanation, when we will draw id, which has already been assigned to some old id, I 'raise' the new id as long as it will be unique in a map. 
Apart from that in Run method there is a method call and the serialization of the obtained result. 
Everything looks nice and neatly. But the watchful eye may notice a problem, namely the use of Newtonsoft.Json library, 
which is accessible from the outside dlls. How to deal with it? How can we say to azure function: "Hey there is a dependency which You should download!"?
In fact, there is nothing simpler, in the folder where is our function, create a file named project.json and add to it the package dependency Newtonsoft. 
In my case file project.json looks as follows:

<script src="https://gist.github.com/MNie/dc4a07ae72081ca6e39cac82e712b35e.js"></script>

Well, we have ready-made code, so at this point all we have to do is a git add/commit/push and wait a few seconds until the source on Azure are updated.
When you update the sources, You can fire a function like that:

https://{appname}.azurewebsites.net/api/{functionName}?data=[432,42]
![result](https://mnie.github.com/img/AzureFunctions/result.png)

As you can see the function operates and returns a result. 
For transferred ids 432 and 42 function generates new identifiers sequentially 8890113132 and 14324747710. 
What's more, we can monitor how often someone tried to call our function with monitoring tools in Monitor tab:

![monitor](https://mnie.github.com/img/AzureFunctions/monitor.png)

It seems that azurefunction is a very powerful tool, eg. for some tasks run successively from time to time. 
With Continuous Integration and integration with version control systems, as well as through a tool to support the monitoring function, so check what request and with which arguments as it ended, finishing on performance machines which checks the performance of our functions. 
Regarding the possibility of writing the azure funtions in F# it is definitely worth to try on Your own and also to wait for the next update, which surely will bring more default triggers and more examples. 
What could help in the creation of functions int F#.

All code could be found [here](https://github.com/MNie/IdsGeneratorAzureFunctions).
Thank you :)