---
layout: post
title: Azure Notebook in F# - creative way to share your notes beside the code.
---

Hey!


In today post I would like to present to you the "new" azure functionality which have been made available for us on December 1 this year. It's about creating notes in the Azure Notebooks in F#. What does it mean and what is it?

Visually, it looks very much like a word / excel online, except that it gives us the opportunity to write/run/show the code, to others by sharing the link. It sounds very intriguing.

Therefore, let's try this functionality! I decided to use [code](https://github.com/MNie/AzureTextAnalysis), which I created before. Thanks to that I will be able to easily compare the results and see if everything works nicely!

So let's get started!

First, we enter the web site: https://notebooks.azure.com/ and log into our azure account.
We create a new library.

![create lib](https://mnie.github.com/img/24-12-2016AzureNotebooks/createLib.png)

We go to the newly created library by clicking on its name. 

![open lib](https://mnie.github.com/img/24-12-2016AzureNotebooks/openLib.png)

At this point, we create a new "notebook".
Click "Open In Jupyter."

![open in jupyter](https://mnie.github.com/img/24-12-2016AzureNotebooks/openJupy.png)

In the right corner, select "New" and select F#.

![new notebook](https://mnie.github.com/img/24-12-2016AzureNotebooks/newNotebook.png)

At this point, we have created a blank notebook in F#, where we can enter pieces of code, or text. I think the best thing will be moving step by step after each of the steps that I made to create final script.

We start by adding, a brief description of Markdown.

![add cell](https://mnie.github.com/img/24-12-2016AzureNotebooks/insertCellabove.png)
![markdown cell](https://mnie.github.com/img/24-12-2016AzureNotebooks/markdowncell1.png)

Well, we have a brief description of what will be in our notebook, let thus loading the nuget packages that will be needed at a later time. We start by adding fields in which we can write in F#. We add cell the same as before, but this time we also set it's type:
 
![set cell type](https://mnie.github.com/img/24-12-2016AzureNotebooks/setCellType.png)

First we want to initialize the packet for our notebook, which shows the following command

<script src="https://gist.github.com/MNie/f884ed41999e93bc51bc473811c7ae37.js"></script>

Ok, we have added the command. The question is what is it doing? Earlier I mentioned that the code can be executed, so to "perform" that line, press "run" in the menu at the top

![run cell](https://mnie.github.com/img/24-12-2016AzureNotebooks/howToRunCell.png)

Since we've added packet at this point, let's get to downloading and installing the nuget packages we need, which is realized in a particular way:

<script src="https://gist.github.com/MNie/432dbf1fba6ba2bea5cb60173e4359b0.js"></script>
![Install nuget](https://mnie.github.com/img/24-12-2016AzureNotebooks/installNuget.png)

If everything went okay, you should see an empty field "out" 

if you commit a mistake in a field out should be a stacktrace of an error.

![error](https://mnie.github.com/img/24-12-2016AzureNotebooks/error1.png)

Since we download packages we could load them to our script by this command:

<script src="https://gist.github.com/MNie/00c2dfed665b6b95b36567c67a70472d.js"></script>

At this point, we can go to our script. As I mentioned at the beginning, I assumed that we are using already "ready" code. However, when writing/rewriting we could seen a few cool features, among other things:

- Intellisense, showing return types and accepted by the function.

![intellisense](https://mnie.github.com/img/24-12-2016AzureNotebooks/intellisense.png)

- Highlighting errors in syntax

![error2](https://mnie.github.com/img/24-12-2016AzureNotebooks/error2.png)

- Well, let's say that our note is ready, we can still do with it?

We can save it and share with someone in the form of a link like this is done with files on onedrive

![share](https://mnie.github.com/img/24-12-2016AzureNotebooks/share.png)

- Insert/embed our notebook as a widget:

![widget](https://mnie.github.com/img/24-12-2016AzureNotebooks/widget.png)

- Download it on the computer in a multitude of different formats:

![download](https://mnie.github.com/img/24-12-2016AzureNotebooks/download.png)

In conclusion, thanks to the azure notebooks we have the opportunity to create notes, together with fragments of code that can be co-created by people with whom we share links. Most importantly, the code is executed, so we can immediately see what the results generates a block of code, also intellisense and highlighting errors in syntax, are a very big plus. The last advantage is the ability to create presentations. I highly recommend you try this functionality :)

Thank you for reading!

You can check my azure notebook in F# [here](https://notebooks.azure.com/library/mnieblog)