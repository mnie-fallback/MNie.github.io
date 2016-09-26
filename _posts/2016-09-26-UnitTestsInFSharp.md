---
layout: post
title: Unit tests in F# using XUnit
---

Hello!

In today's post I would like to raise the subject of unit tests in F#. My programs in F# depends on running some single function without any business logic hidden in the depths. Due to this fact that I had no opportunity to write unit tests for these programs, I decided to see how it looks like. 
So I rewrote tests that wrote once to one of my the students project in C#. Functionality was responsible for generating the correct parameterized query string basin on provided arguments. 
Tests in C# are written using the XUnit and Shoudly libraries. Therefore, when prescribing them to F# I also want to use XUnit library.

So where to start writing tests in F#?

We start by creating a project (project type library) in F#. 
Then install using the Package Manager Console, Xnit, 
<script src="https://gist.github.com/MNie/a143107bfa69c7a328f7f40c4b30c73e.js"></script>
in addition add reference to the project that we want to test. At this moment we have everything ready to start writing test in F#. 
To be able to run tests from Visual Studio, you must install a package called: Xunit.runner.visualstudio
<script src="https://gist.github.com/MNie/df6241330169234beae19d88a625b9c5.js"></script>

We start by adding the following namespaces:

<script src="https://gist.github.com/MNie/a75011ca508cc432b641c74f615ca7fc.js"></script>

Then we create a class (type) which will be responsible for testing one class of our code of business logic.

<script src="https://gist.github.com/MNie/975ddf80e9ff2f9d74c17b771eb0c3af.js"></script>

In the body of our class you will want to write tests. Let's look first how this tests are written in C#

<script src="https://gist.github.com/MNie/d83cb79f143e11e70363d58f36c788d7.js"></script>

We can see the division of the test for sections Arrange/Act/Assert, as well as the name of the test ( "should_create_empty_quert") which looks ugly, due to the fact that later in our test Runner it will be displayed with underscores:

![C# test result](https://mnie.github.com/img/fsharpxunittesting/CSharpresult.png)

Well let's start by rewriting our test to F#. Prescribed test looks like this:

<script src="https://gist.github.com/MNie/e92ed05b21d24efbe95857cec591fa16.js"></script>

As we can see, the test has an annotation similar to that from test written in C#, but very cool thing is, we can name it like in C# (with an underscores), or using dual character '`' to name it as a plain text (with spaces between words) without artificial underscores, which will looks like follows in a test runner:

![F# test result](https://mnie.github.com/img/fsharpxunittesting/FSharpresult.png)

I really like this approach, we don't have unnecessary underscores in test names just simply the name of the test! This is a definite plus for the tests written in F#. We can also see the body of the test, in which we create QueryBuilder object and call on it CreateQuery method and check whether the returned string is empty. 
We could reflect on a better separation of Arrange/Act/Assert sections, although I think that they are visible enough to notice what, where and when happens.

At this point, we are ready to fire test(s). Firing is done by pressing the right mouse button on the test project and then click 'runtests'. To run the tests in a manner similar to the resharper running single tests in C#, we can simply call the function corresponding to the test at the end of the test file, as follows:

![How to run tests](https://mnie.github.com/img/fsharpxunittesting/HowToRun.png)
![Test results](https://mnie.github.com/img/fsharpxunittesting/result.png)

You will notice that the format of tests in F#, does not differ quite strongly from format of tests written in C#. Test results in a test runner also looks the same, what shows following images:
![Error in C#](https://mnie.github.com/img/fsharpxunittesting/CSharpError.png)
![Error in F#](https://mnie.github.com/img/fsharpxunittesting/FSharpError.png)
I think that writing tests in F#, or rewrite them from C# can be a nice way to learn the language as well as a way to gradually introduce it to your own/commercial projects. Of course, the biggest advantage is that you can write functionally, so we have a completely different understanding of the program. 
Very interesting option is also the fact that we can name tests in words (without unnecessary characters underscore).

All code could be found [here](https://github.com/MNie/FSharpUnitTesting).
Thank you :)