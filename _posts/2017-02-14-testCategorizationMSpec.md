---
layout: post
title: Test tagging/categorization in Machine.Specifications
---

Hi!


In today's post I want to say something about tagging/categorizing tests. Which, depends  frameworks where it is implemented in different ways.

I want to focus on tagging tests using Machine.Specifications (through the Subject attribute) and Mspec.Runner as a test runner. As a continous integration system I use TeamCity. So altogether I would like to answer the question whether categorization/tagging is worthy of our attention.


Tagging tests. Should we do it or not?
I assume that as elsewhere, where there are two possibilities, there are also two schools. One says that tagging is bad because the whole test in the output window does not looks like a 'poetry', because it's difficult to call 'poetry' something like this:

<script src="https://gist.github.com/MNie/889de9eb3edc041913cae4151880b847.js"></script>

Definitely better in this case looks simply:

<script src="https://gist.github.com/MNie/22a17d4092bced1601f114b8879a6902.js"></script>

On the other hand we have the information at the beginning of test what class/functionality is tested and we have a possibility to group tests somehow.
That information about what class/function test leads me to 'school' n2.

So how to tag correctly tests in Machine.Specifications?
We should add the attribute "Subject" on the test class that looks like this:

<script src="https://gist.github.com/MNie/5c399d0d81beb476c4b3d72d9ddfd77b.js"></script>

Thanks to that in Mspec.Runner or test report (on TeamCity) we could see a beatiful information about which test was run and whether it passed. This format is as follows:

<script src="https://gist.github.com/MNie/10a241615804f8365c2154972fc3ec80.js"></script>

Are there any problems that tagging tests solves?

Suppose that we have situation like this.
We have two classes that have different tasks (functional) in the same folder.
The behavior of these classes for an empty string as an input is the same, both should return 0.

So the code for these two classes looks as follows:

<script src="https://gist.github.com/MNie/6fd044628a03125d18106830b961615e.js"></script>

The code looks okay, resharper test runner tells us that tests are green, great! We commit our changes to our version control system, we look at our CI system (in this case temcity) and we see a single test.
One test? But we have two tests, riszarper shows green badges, everything should be okay!
What is the problem? Mspec.Runner interprets these tests as the "same" due to the fact that they have the same "name", although it fires them, but they are grouped in a single test. Therefore, on TeamCity the test will have a note

![2 runs](https://mnie.github.com/img/14-02-2017TestCategorization/2runs.png)

How to fix it? By adding a subject attribute to the test classes!
 
But if there are any cons while tagging tests? Well, yes, when you make a mistake mspec runner will not improve for us tagged test and the result for the above mentioned error will be the same.

In summary, I think the option of tagging testing is very helpful and allows you to add another level in the case of grouping tests, especially when the number goes in the number of hundreds/thousands. In addition it allows to more easily locate the possible source of "problems" of red tests.

Thank you for reading :)
