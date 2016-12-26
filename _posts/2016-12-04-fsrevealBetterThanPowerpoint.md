---
layout: post
title: FsReveal as a substitute for PowerPoint
---

Hey,

I suppose that you sometimes have to make a presentation. What do you do? You fires PowerPoint and begin to "write".
While You are "writing" I guess You often complain about it for example that text suddenly begins to overlap with the pictures and so on.
In short all opposed against you?

I would like to present you a different approach to creating a presentation(s). 
The presentation, which can act as a web page. The presentation, which is create in F# language, or Markdown.

Sounds interesting?

Where should we begin to create our presentation?
We download the FsReveal repository from [github](https://github.com/fsprojects/FsReveal/archive/master.zip). 
Go to the folder "slides". 
In this folder you will find files index.md and sample.fsx, which will be modified depending on whether You prefer to create presentation in F# or markdown.
Additionally, there is also a cusom.css file which we can set styles for our presentation and the images folder where we can put all kinds of pictures that you will want to include in the presentation. 
Also how to "run" our presentation? 
Simply type *run build.cmd* in the main folder. 
Default browser should opened and here should be our presentation, if we modify sample.fsx or index.md the changes should be visible immediately in browser.
I will not focus here on the Markdown syntax, or f#, but rather about what "cool" thing gives us FsReveal in comparison to the presentation create in commercial solutions.

What kind of "cool" things mentioned? Among others:

* When you paste a piece of code in F# you can see what is the type of input/output of methods, what are the parameters of the function, etc.

 ![error](https://mnie.github.com/img/2016124FSREVEAL/types.png)

* Our presentation can be hosted as a web page

 ![error](https://mnie.github.com/img/2016124FSREVEAL/site.png)
 [Link to presentation](https://mnie.github.io/FsCheckIntroduction)

* Editing a presentation by a couple of people should not be a problem, because it takes place in an analogous manner to cooperation with any project on github.

* Deploy of our presentation as a web page requires us to do 3 things. 
 The first is to create a repository on github, and the second to modify the file build.fsx so that it led to our repository and our name's on github.
 At the end to type the following command in the console

 <script src="https://gist.github.com/MNie/98aac156a5019879bfa299e68fa5dca7.js"></script>
 <script src="https://gist.github.com/MNie/3ad0f26f2b48459a209e001fd6ca9747.js"></script>

* We can easily manipulate the appearance of our presentation through css styles.

In summary, there is an alternative to PowerPoint, which gives us a wide range of possibilities in terms of modifying the presentation, while not loosing opportunities that give us a well-known commercial solutions. 
Moreover, it seems that we have much more control over our presentation as well as cooperation in its creation seems to be much simpler than in comparison to cooperate in creating a presentation in PowerPoint. 
The last advantage, one of the most important is that FsReveal is free.

Here is a link to my sample presentation about ["introduction to fscheck"](https://mnie.github.io/FsCheckIntroduction)
[Github repository with code](https://github.com/MNie/FsCheckIntroduction/tree/gh-pages)

Thank you for reading :)