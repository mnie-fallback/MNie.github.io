---
layout: post
title: Visual Studio 15.6.0 vs .net like projects
---

Hey,

at the beginning of this week, new Visual Studio 15.6.0 appeared. I decided to update my version to the newest one, because what possibly could go wrong?
Has Microsoft ever released a version of Visual Studio that did not `work` ;)?


Well, while the last dozen of updates did not cause any problems, this update caused a couple of them because Visual Studio starts to hang and shut down after a couple of seconds.
The solution, which was opened, (In fact, an attempt was made to open it) is a solution consisting of about 20 projects, 18 C# and 2 F#.
A quick fix of the sln file that based on excluding F# projects from sln allowed to determine what/who is potentially a problem for the latest version of Visual Studio.
It turned out that F# projects that specify Microsoft.SDK and Fsharp.SDK caused Visual Studio to hang and following information was logged by Visual Studio:

```
LimitedFunctionality
System.AggregateException: Project system data flow 'PhysicalProjectTree Input: 108095' closed because of an exception: System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.AggregateException: One or more errors occurred. ---> System.ArgumentException: An item with the same key has already been added.
   at System.ThrowHelper.ThrowArgumentException(ExceptionResource resource)
   at System.Collections.Generic.Dictionary`2.Insert(TKey key, TValue value, Boolean add)
   at Microsoft.VisualStudio.ProjectSystem.VS.Tree.Order.TreeItemOrderPropertyProvider.CreateOrderedMap(UnconfiguredProject project, IReadOnlyCollection`1 orderedItems)
   at Microsoft.VisualStudio.ProjectSystem.VS.Tree.Order.TreeItemOrderPropertyProvider..ctor(IReadOnlyCollection`1 orderedItems, UnconfiguredProject project)
   at Microsoft.VisualStudio.ProjectSystem.VS.Tree.Order.TreeItemOrderPropertyProviderSource.<LinkExternalInput>b__3_0(IProjectVersionedValue`1 orderedItems)
   at System.Threading.Tasks.Dataflow.TransformBlock`2.ProcessMessage(Func`2 transform, KeyValuePair`2 messageWithId)
   at System.Threading.Tasks.Dataflow.TransformBlock`2.<>c__DisplayClass10.<.ctor>b__3(KeyValuePair`2 messageWithId)
   at System.Threading.Tasks.Dataflow.Internal.TargetCore`1.ProcessMessagesLoopCore()
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
   --- End of inner exception stack trace ---
```

So the files of these projects had to be fixed.
The improvement consisted in swapping the selected SDK at the beginning of the file from:

```xml
<Project Sdk="FSharp.NET.Sdk;Microsoft.NET.Sdk">
```

to:

```xml
<Project Sdk="Microsoft.NET.Sdk">
```

After this amendment I made another attempt to launch the entire solution in Visual Studio. At this point, it was loaded, but the F# projects were not available, only this information appeared:

`project file is incomplete. expected imports are missing`

Dust, delete obj/bin folders for F# projects and C# projects having references for F# projects helped to load the entire solution. Success!


The next step was to build the entire solution. Potentially everything should works fine. Of course, as it happens in such situations, everything did not work.
One of the projects stated that he can not find the dll file of one of the referenced projects. I looked quickly at whether the reference is really added to that project. It turned out that this project has an old csproj file format.
Because of that Visual Studio, shows this information in Solution Explorer:

![properties1](https://mnie.github.com/img/11-03-2018DrunkVs/properties1.png)

and those in the properties of the project:

![properties2](https://mnie.github.com/img/11-03-2018DrunkVs/properties2.png)

For projects with a new csproj file format, all data shown was coherent. So I built the first referenced project, then I built the one that did not want to build a moment ago. It turned out that everything was built correctly.


So it seems that Visual Studio could not keep the order of building projects depending on the references. So I decided that if all projects are in a new format (because it seems that solution which contains a new and old type of cs/fsprojs doesn't works well), this one should not be different so I changed the old format to a new one, which resulted in the fact that all data shown in Visual Studio in the properties of the solution looks right now like that:

![properties4](https://mnie.github.com/img/11-03-2018DrunkVs/properties4.png)

and in Solution Explorer:

![properties3](https://mnie.github.com/img/11-03-2018DrunkVs/properties3.png)

They are coherent. Therefore, there should be no error in compiling the entire solution related to the wrong order of building projects.


In summary, it is often said that we shouldn't start/move project to the technologies/libraries/languages which are not mature enough or stable enough. 
It could be seen that although the subsequent versions of Visual Studio are described as stable, it is better to always have several versions back to the current version.
But this particular situation helps us to spot a weak points of our cs/fsproj files which in fact are incorrect (double sdk, mixing old and new project formats).

Thank you for reading :)