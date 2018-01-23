---
layout: post
title: Resharper TO-DO
---

Hey,

In my daily work, it often happens that with some automatic tests we provide information about user story or defect on which they concern. On daily basis every of my teammates works with Resharper and Visual Studio. So thanks to the TO-DO explorer in Resharper (about which more [here](https://blog.jetbrains.com/dotnet/2018/01/12/linking-external-resources-resharper-items/))). I was able to configure a specific comment format so that I can easily generate links to the application in which our user stories and defects are defined. This application is [RallyDev](https://rally1.rallydev.com).

I start from searching a TO-DO explorer in the Resharper options:

![todo](https://mnie.github.com/img/23-01-2018ResharperToDo/todoexplorer.png)

Then, add new pattern, in my case it will be a patern for a User Story. For a pattern, I define its name, a regular expression that will allow the User Story number to be retrieved. In rallydev it has the form US{numeric id}, for example US111111. I define an url to which I put the extracted information thanks to a regular expression, we can see that in this case I am pulling out the element with name `TAG`. In addition, I want that every pattern/information like that would be shown with an information icon, and the search for these patterns took place only in the comments.

![tag](https://mnie.github.com/img/23-01-2018ResharperToDo/tag.png)

I'm clicking ok. Saves patern(s) for the entire project and check-in changes

![as](https://mnie.github.com/img/23-01-2018ResharperToDo/as.png)

Then I write an example comment and looks whether it works

![runrun](https://mnie.github.com/img/23-01-2018ResharperToDo/run.gif)

Summing up thanks to this mechanism, we are able to provide additional information about User Story/Defect near an automated test with a direct link to a RallyDev app.

Thank you for your attention!