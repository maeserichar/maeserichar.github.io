---
layout: post
title: "Build scripts with Cake"
description: "How to use Cake to develop build scripts for your dotnet applications."
comments: false
keyword: dotnet cake build
---
One of the "bad" things that the people that came from a pure Microsoft develop environment have is that we tend to left the process of building our code in our local machine to the MSBuild script (well, the solution file, really). If you had been so lucky to need to customize that script, you will agree with me that that is a far from pleasant task, as the MSBuild syntax is a bit cumbersome, and excessively verbose, as any good old XML based format.

In the F# world, exists Fake (https://fake.build/), a DSL (Domain Specific Language) for task execution and automation, used to develop builds scripts using F#. For the people more used to C#, there is an equivalent tool, called Cake (https://cakebuild.net/), the one we are going to talk about here.

## Cake basics

There are three (well, four) files that you need in order to use Cake in your project:

* build.ps1 and build.sh: This is the bootstrap script, that ensures that Cake and the dependencies needed are downloaded. It also executes Cake. This is an optional file, you can download and execute Cake by hand if you prefer to.

* tools/packages.config: This file is the equivalent to the packages.config of any C# project, as Cake uses Nuget to manage its dependencies (test runners, ILMerge, etc).

* build.cake: This is your build script. Here you define the different tasks of your build scripts and the dependencies between them. This file is the one we are going to spend most of our time on.

In order to initialize your repository, you can use the following commands:

* In Windows:

  ```powershell
  Invoke-WebRequest https://cakebuild.net/download/bootstrapper/windows -OutFile build.ps1
  ```

* In Linux/OS X

  ```bash
  curl -Lsfo build.sh https://cakebuild.net/download/bootstrapper/[linux|osx]
  ```

This will download the needed files, and once this is done, if you want to execute your build script, you only have to launch the appropriate script for your OS:

* In Windows:

  ```powershell
  .\build.ps1
  ```

* In Linux

  ```bash
  ./build.sh
  ```

## Tasks

As we said in the previous section, our build script is composed by tasks. A task define a piece of work, something that you want to be done in your build script.

The minimum elements of a task are a name and the action to be done:

```C#
Task("My task name")
    .Does(() => {
        // Here comes the code of the task, for example:
        DotNetCoreBuild("./MySolutionFile.sln");
    });
```

In case we had any task that depends on another task (for example, we need to build our solution before running our test suit), we can mark it with `IsDependentOn`:

```C#
Task("Build")
  .Does(() =>
{
  DotNetCoreBuild("./MySolutionFile.sln");
});

Task("Test")
  .IsDependentOn("Build")
  .Does(() =>
{
  DotNetCoreVSTest("PathToMyTestsDlls");
});
```

## Imports

Maybe, you need to use an external tool for one of your tasks, as for example getting a test coverage report using OpenCover. One option is to have that tool installed on the machines where you (or your collaborators) want to build the solution, effectively adding a pre-requirement for you project to be built. But, luckily for us, Cake allows you to define your dependencies as pre-processor directives and it will be downloaded using Nuget when your script is executed.

In this sample, we are importing OpenCover and ReportGenerator to get test coverage data and generate an HTML report of it:

```C#
#tool "nuget:?package=OpenCover"
#tool "nuget:?package=ReportGenerator"

...C#
Task("Coverage")
  .IsDependentOn("Build")
  .Does(() =>
  {
  var openCoverSettings = new OpenCoverSettings
    {
        OldStyle = true
    }
    .WithFilter("+[*]* -[moq*]*");
  OpenCover(tool => {
  tool.DotNetCoreVSTest("./**/bin/**/*.Tests.dll"); 
  },
  new FilePath("./result.xml"),
  openCoverSettings);

  ReportGenerator("./result.xml", "./report/");
  });
...

```

## Input

As part of you build script you may need to pass some input data, maybe because you don't want to commit it to the build script (for example your nuget API key). For that, you can use custom arguments that can be passed to the execution of cake. 

First, you define a variable to hold your arguments:

```C#
var apiKey = Argument("apiKey", (string)null);
```

And then, you can use it in your code as a normal variable:

```C#
Task("Publish")
  .IsDependentOn("Test")
  .Does(() =>
  {
    var package = GetFiles(@"pathToYourNugetPackage");

     NuGetPush(package, new NuGetPushSettings {
       Source = "https://api.nuget.org/v3/index.json",
       ApiKey = apiKey
     });
  });
```

## C# code

One of the best things about Cake (or at least if you are familiar with C#) is that at the end of the day, all the code inside your Cake script is C# code, so you are not limited to the Cake DSL, you can use any of your C# "magic" on it. So, lets said, in the previous example, you want to provide a meaningful error message if anybody try to push a new version of the Nuget package without providing an API key:

```C#
if (string.IsNullOrWhiteSpace(apiKey))
{
  throw new ArgumentException("For the 'Publish' target you must provide an API key.");
}
```

## VS tooling

There is an extension available for Visual Studio, called *Cake for Visual Studio*, that provides syntax highlight for Cake scripts, but at the moment there is no support for intellisense. Also, if your build script is part of your solution, your tasks would appear on the *Task runner* window, where you can run them.
