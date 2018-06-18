---
layout: post
title: Rider case sensitiveness
---

Hi,

a couple of days ago I started (in fact I come back) using a Rider IDE when developing C# based project. I came to a weird situation when trying to run unit and integration tests, cause Rider to start them, but it couldn't discover tests in one project. It seems weird, cause as I remember everything works fine with discovering those tests in Visual Studio or in a Cake script. I start digging into what could be wrong here. I thought that maybe there could be some problem in a csproj file which causes Rider to not discovered tests properly. I find out that the csproj for this project which couldn't be discover by Rider test runner (but could be by Resharper test runner) looks like this:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netcoreapp2.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="autofac" Version="4.8.1" />
    <PackageReference Include="Dapper" Version="1.50.4" />
    <PackageReference Include="FakeItEasy" Version="4.5.1" />
    <PackageReference Include="fluentassertions" Version="5.3.0" />
    <PackageReference Include="microsoft.extensions.configuration" Version="2.0.2" />
    <PackageReference Include="microsoft.extensions.configuration.json" Version="2.0.2" />
    <PackageReference Include="microsoft.net.test.sdk" Version="15.7.0" />
    <PackageReference Include="NUnit" Version="3.10.1" />
    <PackageReference Include="NUnit3TestAdapter" Version="3.10.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\SomeRelatedProject.csproj" />
  </ItemGroup>
</Project>
```

At first glance, everything looks fine. I compare it with another csproj for a project for which tests were discovered correctly and it looks like this:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netcoreapp2.0</TargetFramework>
    <AssemblyName>JetShop.Commerce.IntegrationTests</AssemblyName>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Dapper" Version="1.50.4" />
    <PackageReference Include="FakeItEasy" Version="4.5.1" />
    <PackageReference Include="FluentAssertions" Version="5.3.0" />
    <PackageReference Include="FsCheck" Version="2.10.10" />
    <PackageReference Include="FsCheck.Nunit" Version="2.10.10" />
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="2.0.2" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="2.0.2" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="15.7.0" />
    <PackageReference Include="NUnit" Version="3.10.1" />
    <PackageReference Include="NUnit3TestAdapter" Version="3.10.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\RelatedProject1.csproj" />
    <ProjectReference Include="..\RelatedProject2.csproj" />
  </ItemGroup>
</Project>
```

Yup this is it, I got you! It seems that rider decides if he has to scan some specific project(s) for tests only when this project contains in dependencies a `Microsoft.Net.Test.Sdk` but as it seems when rider scans csprojs it scans them taking into consideration a case sensitiveness, so when I change a `microsoft.net.test.sdk` to a `Microsoft.Net.Test.Sdk`, everything starts to work correctly.

So when you are using rider please be cautious that it could be case sensitive sometimes :)

#edit
It is already [fixed](https://youtrack.jetbrains.com/oauth?state=%2Fissue%2FRIDER-16914) and should work correctly in Rider 2018.2.

Thanks for reading!