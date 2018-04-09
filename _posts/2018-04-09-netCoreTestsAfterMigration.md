---
layout: post
title: Running tests via Resharper after migration to a new project file format
---

Hey,

some time ago I converted project files to a new format what causes some problems in tests projects. For unit tests we use Machine.Specifications. For running specs we use Resharper, so the only external dependencies in tests are Machine.Specifications and Rhino.Mocks.

As I already mention, there are some problems. The biggest one was a discovery of unit tests, which in fact stops working. After building a project in context menu there wasn't an option to run unit tests, and when we open some file with unit tests, tests only in this files magically shows up in unit test explorer, and run tests option in context menu appears. After opening subsequent files, subsequent tests were discovered.  Running tests via console runner mspec-clr works fine. So I suspected that it is a Resharper fault.

It turned out that there is an information in Resharper logs, that test adapter is missing in bin/net462 folder near a tested DLL file. So I think, that there might be some lacks in a project file. So I add two following NuGet packages:
1. [Machine.Specifications.VisualStudio.Runner](https://www.nuget.org/packages/Machine.Specifications.Runner.VisualStudio/)
2. [Microsoft.NET.Test.Sdk](https://www.nuget.org/packages/Microsoft.NET.Test.Sdk/)

The first one is used to running Machine.Specifications tests throw a Visual Studio Test Explorer. The second one is eligible for running tests in .net core projects, but as it shows not only in .net core projects but in all projects which have a new project file format.

Thanks to that fix, an option to run all tests in a project is one more time available, the same with discovering tests in the whole project (it simply starts working). So to sum up we have to remember that there are some additional things to do after migration to a new file format, like referencing additional libraries in case of tests projects.


Thanks for reading!