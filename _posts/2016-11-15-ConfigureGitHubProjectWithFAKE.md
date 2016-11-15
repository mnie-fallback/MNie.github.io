---
layout: post
title: How to configure Your open source project on github with FAKE
---

Hey,

 
We often create/contribute to the open-source projects. There comes a time that we want to make that claim, or to "give" project to a "community", to help us in its implementation. 
In further improving and developing it.
What to do in such a case? Where to make the project so that others could contribute to it? The answer to this question in 99% of cases will be a "github".
We create repository on a github, we put our code there, the community start contributing. But with the time going.. Do we want everyone to have the opportunity to throw something into the main "branch"? 
Probably not. We "apply" in this case, pull requests, but how do we know that the code that someone is trying to checkin definitely works? 
How do we know that we keep continous integration? 
And that's where comes to us travis, which is a free CI for github projects and FAKE, so that we can provide the "continous integration" of our project.
Great!
But where to start, what and how to write?
As I said, assume that we had repository in github right now, we login to travis-ci.org using our github account. 
In the settings (travis-ci.org/profile/{yourName}), we switch the "switch" button telling us that we want our project to be "turned on" in Travis.

![switch](https://mnie.github.com/img/20161115githubtravisfake/switch.png)

Okay, "we turned on" our project in Travis. 
But what does that do? I do not see the project suddenly began to build. 
What we have to do? You must add an empty file named: .travis.yaml, to the root of your repository. 
For now, we want to only build the project. What we have to write in the previously added file (.travis.yml)?

<script src="https://gist.github.com/MNie/23c07ebecc776acf52f84f352c1c1d68.js"></script>

So going throught of each parameters, the language is defined as csharp, for projects written in C#/F#/VB. 
Solution is a sln file. In step install travis perform operations associated with preparing everything for building/releasing tests, etc. In our case, we restore nuget packages for our project. 
In step script, travis fired all kind of scripts, and in our case it is the command for building a project in the release configuration.
Well we have a building script right now, but where is the FAKE which is mentioned in the title? 
Well, we come to the final version of the Travis (yml) script, which will be running another script (FAKE script). 
Which will have the task to build the project and then run all tests.
File .travis.yml looks at this point like this:

<script src="https://gist.github.com/MNie/a05fbd2a223681a465110157a6789d37.js"></script>

As we can see, there are some changes in a script, in addition to nuget packages restore, we add commands to install the FAKE, which we will need to launch our script (FAKE script). 
The last point is to install xunit runner package to launch project tests (in this case FsCheck specs). 
Let us now turn to FAKE script.

<script src="https://gist.github.com/MNie/5d9b6026d77b1f3648679d1f05c49784.js"></script>

So yes, we are going from the very beginning, starting from referencing/opening libraries, which will be used. 
Then, we determine where our tests are. In our case they will be located in every dlls with name test and in a folder test. 
Then we set our directory to the source project directory. In the next point we define the next "step" ("targets") of a build script. 
The first is the "Clean", which is responsible for clearing bin and temp folders. 
Another "Build" is responsible for rebuilding project, if it exists. 
"RunTests" while is responsible for the firing of all the tests if such tests exists. 
At the end we specify order of "targets"/steps in a script, so we start from step "Clean", then "Build" and end with "RunTests". 
At the end we "fire up" build script with the command "runtargetordefault". 
The effect of the script is shown in the picture below:

![ok](https://mnie.github.com/img/20161115githubtravisfake/ok.png)

Build script steps are marked in green. 
Each of the steps as shown in the photo, or on the website(link) is logged nicely, so we have insight into what is currently being done, eg. How the test are fired, what package is pulled etc. 
Regarding the same steps of the script that builds when something goes wrong, this step is colored red instead of green as shown in the following picture / link.

![error](https://mnie.github.com/img/20161115githubtravisfake/error.png)

As we can see. We build a CI solution for project in a simple and quick way, by Travis and Fake. 
So you do not have to worry about whether our project "works" properly. When someone adds new features to project, or wants to add it.
In order to further expand the script and writing/firing more advanced things I recommend You to read a fantastic FAKE documentation, which is available [here](http://fsharp.github.io/FAKE/gettingstarted.html).


* [repository](https://github.com/MNie/DateSeqGenerator).
* [green build](https://travis-ci.org/MNie/DateSeqGenerator/builds/174579724).
* [red build](https://travis-ci.org/MNie/DateSeqGenerator/builds/174566325).

Thank you for reading!