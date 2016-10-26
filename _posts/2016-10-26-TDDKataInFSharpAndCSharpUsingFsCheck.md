---
layout: post
title: TDD Kata in F#/C# using FsCheck
---

Hey,

Browsing Roy Osherove's blog I saw a short kata tasks designed to test in practice TDD approach to software development. 
The purpose of this kata was to create a simple calculator using TDD. 
I thought it would be a nice idea to create and describe the solutions of this Kata by FSChecka, xUnit and C# and F#. 
Why I think it might be fun? As a rule, such solutions can be found in C# with usage of NUnit or xUnit, and here we will try an entirely different approach to writing unit tests. at any rate unit.

The aim of this Kata will be to create a "calculator". 
Let's start with the steps that should be implemented:
1. Create a class Calculator, which will include the Add method. 
	The method should accept a single parameter of type string.
	a. We start from the fact that passing an empty string to a method should return 0.
	b. For single argument should return numeric value of this exact parameter, otherwise it should return 0.
2. Function (add) should be converted in such a way that it can accept multiple arguments that the input string will be separated by a space.
3. Our method should also be able to separate the input arguments with the newline character '\ n'.
4. We need to modify our method so that it will accept another argument. 
	This argument will tell our function how the input delimiter looks like. Delimiter should be of type string.
5. The next step is to modify the method so that it threw an exception when input argument will contain any negative numbers.
6. The last task is that, the numbers greater than 1000 should be skipped during summation.

If you want to do this kata on Your own, [here is an empty project](https://github.com/MNie/TDDKataFirst/tree/kataStepByStep) with a list of tests to implement.

Here is my resolution :)
Well, since we have written in paragraphs what we should do, 
let's start with the realization of the first point. To do this, we start by writing a test that will check whether the method add for an empty input string returns 0. 
In this case, I write the normal xUnit specs. This tests are as follows:

C#:
<script src="https://gist.github.com/MNie/d503caf17d8289d97165627144fcb651.js"></script>
F#:
<script src="https://gist.github.com/MNie/7a6bd12824859742c934cfd44d014c90.js"></script>

Since we have a tests, we can run them, of course, the tests should be 'red'.

![error](https://mnie.github.com/img/TDDKata1/failed1fs.png)

Due to the fact that the implementation of add method is empty, let's implement it. 
For now, for any input parameter it should return 0.

<script src="https://gist.github.com/MNie/095dce0e73d07c2eb785b0a50523a1aa.js"></script>

We run the tests again, and as we can see they are green.

![ok](https://mnie.github.com/img/TDDKata1/1fs.png)

So let's go to the point 1b, this time if an input argument has a numeric value it should be return. Otherwise, we should return 0.
To do this, I wrote FsCheck test. 
Why? It will allow us to test the method on a number of input data, and not just a single "input". 
As I mentioned earlier FsCheck test run test for various inputs 100 times (this is the default value, which can be changed in the custom configuration, about which I say more in point 4). 
The tests are below (as we can see, to run/annotate FsCheck test we use an xUnit (annotation on the method [Fact])):

C#:
<script src="https://gist.github.com/MNie/ad798ea3b36bd0c5f86f5f68f7d137c3.js"></script>
F#:
<script src="https://gist.github.com/MNie/cb1a7596d2fda1bf5b208fe930b97431.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed1bfs.png)

Before we get to the results of the test, we need to clarify some issues. 
We'll start with the code in C#. 
Prop.ForAll means that test run for all data submitted in 1 argument (which could be read "for all properties do something"), 
for which we do the check referred to in the second argument. 
Worth noting here is also a way of generating input data, which looks as follows:

<script src="https://gist.github.com/MNie/3ea376fddc988d831f8bbb5347b3531d.js"></script>

We can see that we generate any numeric value int and then convert it to a string to fit the input function. 
The Arbitrary type means an instance that wraps the test data generator and shrinker (which is responsible for matching the data that don't meet the condition of our check, and 'shrinks' it to the 'smallest' one which is not passing the test). 
In contrast, Arb.Generate <Titem> is responsible for generating the data type Titem, in this case (test 1b), they are integers.
We also could write C# code like this:

<script src="https://gist.github.com/MNie/552cf544a57ea2715594e3a948e593bc.js"></script>

But if we have more complex models generic version of Prop.All would not works.
In F# we define parameters to a function with a valid types and FsCheck do all for us.

Now we proceed to implement the Add method. 
For each of the input, method should return its numeric value. 
Please note that the input argument to the method is a string, so we have to check if the parameter is correct. 
Implementation of the Add method in this stage looks like this:

<script src="https://gist.github.com/MNie/77e54357ca85a7c29d7c9745c1582491.js"></script>

![ok](https://mnie.github.com/img/TDDKata1/1bfs.png)

The next step will be to adapt the add method in that way it could be able to accept many numbers separated by a space as an argument. 
For this purpose, we modified/create new one test methods to accept a list of numbers, which then are combined to a string and pass to the method. 
The tests are as follows:

C#:
<script src="https://gist.github.com/MNie/c999b2c04897a1807dc5ec8de8d2872f.js"></script>
F#:
<script src="https://gist.github.com/MNie/8f555fdedd682643cf399c7bd5bddda4.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed2fs.png)

We can see that in F# case I used several functions that were not used previously. So their and other utils functions implementations are shown below:

<script src="https://gist.github.com/MNie/a551a4ce618fc3a980f9880245ce33e2.js"></script>

Modified Add method looks as follows:

<script src="https://gist.github.com/MNie/578782d498daaeb10e83a42c5b58e936.js"></script>

![ok](https://mnie.github.com/img/TDDKata1/2fs.png)

So far, the number could be separated only by a space, 
we introduce a modification that numbers could be also splitted by a new line character.

Tests looks like this:

C#:
<script src="https://gist.github.com/MNie/8105ec758e73c61471304ffe96ac78fe.js"></script>
F#:
<script src="https://gist.github.com/MNie/cb4e17e1e9433e093e31aa68db576315.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed3fs.png)

The modified code of an Add method:

<script src="https://gist.github.com/MNie/1279550c2863685a937f0e3f244ab122.js"></script>

![ok](https://mnie.github.com/img/TDDKata1/3fs.png)

Another point is to change the function so that it is able to accept an additional argument, 
so we could call a function with any delimiter. In C# case, you must write a custom data generator. 
So that each test was passed object with a list of numbers and a custom delimiter, which is in the form of a single string.
Such generator looks like this:

<script src="https://gist.github.com/MNie/2e095b640465c135af1e8adfffa7a7bc.js"></script>

The modified tests is as follows:

C#:
<script src="https://gist.github.com/MNie/a84e159053a5c6c5d43622d8f7d50fbb.js"></script>
F#:
<script src="https://gist.github.com/MNie/68be9f5bdfea5470f52d01c538411d1d.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed4fs.png)

Modified add method looks like this:

<script src="https://gist.github.com/MNie/8494ea00aa52c085e2948cc649b17523.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed4fs.png)

We find, however, that the test is not green, why is this happening?
Delimiter can be any string of characters, and thus the number. 
In the case when it is a number, and exactly the same number(s) exists in the input data array. The result of a method is incorrect. So what we have to do? 
We must bear in mind that the delimiter may not be a number, in this case, we modify our F# test and C# generator as follows:

C# generator:
<script src="https://gist.github.com/MNie/0454f894ec818371b8d49c941001782e.js"></script>
F# test:
<script src="https://gist.github.com/MNie/8ffad1be24f227e8ed5ca9afd3c0525b.js"></script>

![ok](https://mnie.github.com/img/TDDKata1/4fs.png)

As mentioned at point 2. 
I wanted to mention here how to modify the test run configuration. 
We catch an error just because test run on various data a few times. 
What to do to avoid this? You can specify a modified configuration. It looks as follows:

<script src="https://gist.github.com/MNie/b07465bdb607b2eb3833e6de88e04fc3.js"></script>

Parameters StartSize and EndSize define the granularity at which data generator generates data. 
Generators in FsCheck increase their value in small steps, eg. 1, 2, 4, 10 ... etc. 
An important aspect is also the fact that the launch of the test with their own setup is a little different, instead Check.Quick(test) or Check.QuickThrowOnFailure(test) we call Check.One(config, test).

Penultimate thing to do is to throw an exception if we pass a negative numbers to a add method.
So here are the tests:

C#:
<script src="https://gist.github.com/MNie/0cc58d4d40bc921974787e38a3dc7fab.js"></script>
F#:
<script src="https://gist.github.com/MNie/1e8e452d7140398daf109d8fa40eda52.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed5fs.png)

Modified Add method looks like this:

<script src="https://gist.github.com/MNie/508cdd571433c28ef92b31ce2a3090eb.js"></script>

![ok](https://mnie.github.com/img/TDDKata1/5fs.png)

The last point to achieve is to skip adding up numbers that are greater than 1000, so that eg. An attempt to call the Add method with two values 1001 and 1 should return 1.
Test code looks like this:

C#:
<script src="https://gist.github.com/MNie/1ca28fa4c27a58c83c54b98e13b87152.js"></script>
F#:
<script src="https://gist.github.com/MNie/ecf01e3648175f2ab63bc052b697a8b3.js"></script>

![error](https://mnie.github.com/img/TDDKata1/failed6fs.png)

Add method:

<script src="https://gist.github.com/MNie/04d592bfbc45208d05e3ccd53f6cbca8.js"></script>

![ok](https://mnie.github.com/img/TDDKata1/6fs.png)

![ok](https://mnie.github.com/img/TDDKata1/all.png)

In summary due FsCheck we have the opportunity to test the reaction of the data for a large range input.
What we would not be able to fully check via normal unit testing eg. In NUnit or xUnit (which by the way have attributes to fire tests with specific parameters TestCase for NUnit and Inline for xUnit). 
With random testing we could draw a conclusion about delimiters that they can not be a number what could be impossible to achieve with conventional unit tests. 
The mere use of FsCheck with such a problem may seem like rifle shooting to the fly, but it has to show us the basic features and capabilities of property-based testingu and random testing using FsCheck, which has the intention to set/check properties of arguments/property.

Thanks for reading.
All source could be found on github:

[All tests](https://github.com/MNie/TDDKataFirst/tree/allTests).
[Final Solution](https://github.com/MNie/TDDKataFirst).
[Kata step by step](https://github.com/MNie/TDDKataFirst/tree/kataStepByStep).