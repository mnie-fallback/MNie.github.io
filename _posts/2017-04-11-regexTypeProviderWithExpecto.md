---
layout: post
title: Expecto and RegexTypeProvider in action
---

Hi,

some time ago I had a task at work to give developer ability to define some dependencies between data send by user (selected filters on site or something) and the configuration of data displayed on page (maps, charts etc.) 
Because I work in C # on a daily basis. The solution was based on this language, Regex and the tests written in Machine.Specifications. 
Of course, my question was, is it possible to do it in a cooler/simpler way in F #, using some other libraries and of course using functional approach?
So I decided to do the same task (specifically a part of it) using F#, Expecto and RegexTypeProvider, because why not?

Let's start by creating empty projects, in this case it will be a library in F# and a project with tests in Expecto. We add projects by typing *ctrl + shift + p* in vs code and then type *new F# project* and select proper option:

![project](https://mnie.github.com/img/11-04-2017ExpectAndRegexTypeProvider/newproject.png)

Well, we have everything ready for work, so let's start by defining the requirements of this "task."
We assume that by some expression, we will be able to get some values. It's best to illustrate the following:


* By writing "45", I expect that the result the function will be 45. 
* By writing "{FROM_SOME: 467}" I expect the value to be the value for id 467 from some properties/dictionary. 
* By "{FROM_SOME: {FROM_SOME: 467}}" I expect the result to be the value for the parameter obtained from the value for parameter 467. 

It is important to ensure that the "from_some" function will be recursive and loosely defined values ​​are supported.
For this purpose I wrote 3 test cases which should cover the above cases.
These tests look like this:

<script src="https://gist.github.com/MNie/45a6385d85be0b0a404b81dec080b7d2.js"></script>

As you can see, I assumed that the "metalanguage" resolving method would take 2 parameters. The first argument is a value to parse of type string, while the second parameter is a dictionary of properties needed to resolve specific values.
I started by creating the same function signature that will always return the first passed parameter.

<script src="https://gist.github.com/MNie/3f38272e62289d554fc6e13fd038fe26.js"></script>

Right now we run the expecto watch mode, so after every build of solution we could see if tests passed or not. How we can achieve that?

![watchmode](https://mnie.github.com/img/11-04-2017ExpectAndRegexTypeProvider/watchmode.png)

In the console we immediately see the result:

![result](https://mnie.github.com/img/11-04-2017ExpectAndRegexTypeProvider/result1.png)


Well, tests will be fired every time you save a file, what you can see in the console (output in vs code).
So we go to the implementation of the "Provide" method. As I mentioned at the beginning, the main task I had was to get to know the RegexTypeProvider. 
So we start by creating a *provider* that we will need, we accomplish this by declaring the type:

<script src="https://gist.github.com/MNie/1ab3285560beb0e6a3ba7627d4fdcea8.js"></script>

We have a type, how can we use it?
In such a simple way:

<script src="https://gist.github.com/MNie/d54a15ebf75d74718e1dddae0819b927.js"></script>

![regex](https://mnie.github.com/img/11-04-2017ExpectAndRegexTypeProvider/regex.png)

What if we wanted to pick first item from the group in our *match*? Nothing easier, we use property "1".
So if input string looks like this: "dedede {FROM_SOME: 34}" then:

* *someRegex().TypedMatch(value)."1".Value* will return 34
* *someRegex().TypedMatch(value).Hit.Value* will return {FROM_SOME:34}

When we know how it works and how it looks, we can go through the implementation of the function itself.
For this I wrote the `getValue` method, which is invoked in the `provide` method:

<script src="https://gist.github.com/MNie/b45ec162e0d0183b771b54c01a9330b7.js"></script>

The `getValue` method is used to derive the value from passed value and properties. You can see here the use of `IsSome`, `extractSome`, `getValueFromProperties` methods how they look?

<script src="https://gist.github.com/MNie/5e3790c1947a216c2e2825e5cd04fd40.js"></script>

`IsSome` is an active pattern that is responsible for returning the resolved value from a passed string with help of *RegexTypeProvider*, or an empty result.
`GetValueFromProperties` is supposed to extract the value from the properties dictionary if there is a value under the appropriate key. Otherwise it just returns a key.
`ExtractSome` is responsible for calling the `getValueFromProperties` method to retrieve the value for the given key and return the result.
The implementation looks like we are ready to save the file and take a look at the result of Expecto:

![result2](https://mnie.github.com/img/11-04-2017ExpectAndRegexTypeProvider/result2.png)

There is still one test which fails. A test that requires a recursive calls. So we add a recursion to the solution. How to accomplish that?
We only modify the `getValue` method as follows:

<script src="https://gist.github.com/MNie/21a1ee0867342f7d161b7dd09094cf3f.js"></script>

We precede it with the *rec* keyword and modify the behavior. If we hit a pattern in our input string we call the function again. 
It can be seen that this change was not too time consuming nor difficult. We save the file and:

![result3](https://mnie.github.com/img/11-04-2017ExpectAndRegexTypeProvider/result3.png)

Everything is fine.

In conclusion, by using F#, I can conclude that the amout of lines is much more less than in the equivalent c# solution is about ~40 lines of code versus ~100 lines in C#. 
However, the biggest plus that I noticed in F# solution compared to the solution in C# is the ease of modifying the method to work recursively. 
I know that implementing this in C# has caused me many problems, but in F# it was pretty straight forward. 
I also liked working with the RegexTypeProvider, where I can determine under which property there will be my *match*.
Also Expecto watch mode works great, thanks to which I knew immediately whether if my solution was working or if there is still some work to do.

Thanks for reading!

* [Source](https://github.com/MNie/Extractor)
