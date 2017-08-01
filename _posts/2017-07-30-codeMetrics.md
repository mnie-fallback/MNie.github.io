---
layout: post
title: Decrease technical debt with help of CodeMetrics
---

Hi,

By doing my daily work, I try my best, so that code which I produce is of the highest quality. It is important that it could be easily tested and managed. However, it must be borne in mind that the projects we do at work are not individual tasks, and dozens of people are working on them. However, it is well known that everyone can have a bad day that will cause a bad code to be generated that will lead to headaches when we look at it after some time.

How can we control the quality of code that is produced in our team? The solution could be the code metrics available in Visual Studio, which allows us to statically analyze the code in terms of 5 parameters.
These parameters are:

* Cyclomatic complexity - the complexity of branching methods in code;
* Maintainability index - measures ease of code management;
* Depth of inheritance;
* Class coupling - an index that corresponds to the relationship between classes;
* Lines of code - executable code lines.

The way to improve each of these metrics is similar, just to improve the code (simple right ;)?). In this post I will focus on cyclomatic complexity. Due to the desire to decrease code debt in a project. At the same time without throwing suddenly at all the bad places. I decided that the best way is to divide the code due to 3 parameters.

The first one is already mentioned before cyclomatic complexity, the second is information about the last modification date, the last number of changes made in a given class (in the project in which I work, each class is in a separate file).
Having already an idea about on what data I want to operate, I had to go to get them. Code metrics can be get by exporting project statistics to a .csv file. We start by calculating code metrics. By right-clicking on the project and selecting from the context menu "Calculate Code Metrics"

![project](https://mnie.github.com/img/2017-07-30CodeMetrics/run.jpg)

After generating the statistics, we are going to export them to excel.

![project](https://mnie.github.com/img/2017-07-30CodeMetrics/result.jpg)

By clicking the green excel icon we can filter the exported data so that they contain only information about the projects/methods/namespaces that we want to analyze.

By going to the date of the last modification of the file, and the number of modifications to the file, I used 2 git commands that look like this:

<script src="https://gist.github.com/MNie/91bd2beabbf872e9db34d1d7b96cf327.js"></script>

<script src="https://gist.github.com/MNie/e92a9570fd2154d7b6c04a45cb5fc841.js"></script>

At this stage I had the data needed for the analysis. I need to determine which methods should be improved and on what values of metrics it should be determined. I decided that it was best to deal with functions that are often modified, modified recently, and have a high complexity as I said before. So I took the approach that I want to fetch methods that have the following attribute values:

* Cyclomatic complexity> = 5
* Last modification time <= 4 weeks = 28 days
* Moderation rate = 5

To achieve my goal I wrote a piece of F # program/script that uses the benefits of [CsvProvider](http://fsharp.github.io/FSharp.Data/library/CsvProvider.html). I started by declaring the types that represent the data I have:

<script src="https://gist.github.com/MNie/ea7cd69e6bdf4fa3ef745aa35c766d11.js"></script>

Then I created a type that will reflect the resulting file with methods that require changes:

<script src="https://gist.github.com/MNie/89e086f0201b407df24cb2069e0ec5bb.js"></script>

I'm interested in the name of the method that needs a change, all the attributes that decide that this method should be changed and where the file is located. What is interesting, that I use the "StructuredFormatDisplay" attribute, which allows you to overload the object's "print" method while using "sprintf"% A "ourObject". Thus, in the code responsible for the data connection, there will be no logic associated with the way of how the Info class should be displayed.

By going to the executable code, I started by loading the data and set the limit value for "4 weeks back"

<script src="https://gist.github.com/MNie/cb94c28a04c7795541c0ed1cf8d5da43.js"></script>

As I said before, I'm only interested in methods and I want to have information about them, so I wrote following piece of code:

<script src="https://gist.github.com/MNie/16065ba51e795acab983a48b1acd500a.js"></script>

It has the task of loading the data by proposing code metrics, going through all the lines, and filtering only the methods. Then it maps everything to the Info type. You can see here the use of the fileInformation function. We could right now based on variables names that this method returns the name of the file in which the method is located, the date of the last modification of the file, and the number of changes made to it.

This method looks like this:

<script src="https://gist.github.com/MNie/a2599e4b2402a67e37724bd339be25f3.js"></script>

For a given class, we declare a local function byName, which searches for files that end with the class name and the .cs extension because we are interested in C # files. Then we find the record corresponding to the file name in the data regarding the date of the last modification and the number of changes. Then we declare 3 variables, where the first contains the file name, or the value "File not found" when the file was not found, lastChange contains the date of the last modification, or the minimum date (DateTime.MinValue), and numberOfChanges contains the number File modification or 0 if no matching record was found. At the end, the tuple containing the three variables is returned.

Having data in this form, the next step was to filter it out based on attributes that were listed earlier with the corresponding values ​​(complexity == 5, lastEdit >= fourWeeksAgo, howManyChanges >= 5), then sorting the results descending by complexity and write result to the file so I could share this information also within the team.
The above steps were done using the following code:

<script src="https://gist.github.com/MNie/be2210af500ef9b9e46c3802615c0263.js"></script>

In summary, with tools available for C # developers (using Visual Studio), Code Metrics and Git's console, we can somehow monitor the quality of our code and catch bits of code that should be repaired to make the product easier to manage and less complex for developers. By means of F # and DataProviders, we could see that F # also fulfills the role of scripting language in which we can easily process data sets to extract the most interesting information.

Thanks for reading!

* [Source](https://github.com/MNie/CodeMetrics)
