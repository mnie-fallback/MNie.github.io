---
layout: post
title: Deleting old branches using Azure Functions
---

Hi,

In every project which I worked, I encounter the same problem. After some time there were a lot of branches on the origin. That causes a not necessary use of space and problems when searching for the right branch. Because of that, I decided to write a utility tool which would fix that issue.

The main purpose of the application was to delete regularly all branches from all projects and repositories in our AzureDevops space. Those branches have to meet the following requirements to be deleted:
- a branch should be older than 1 month;
- a branch should not have any open pull requests;
- the branch name is not equal to test, nor beta, nor release.

One of the requirements was to run the code responsible for deletion periodically. I decided to create an azure function in F# which would be triggered by time. The time when the function would be trigger would be a 6 am every day.

I started by creating an Azure function of type TimeTrigger, but it is not so obvious to create a function like that in F#. First of all, you have to create a TimeTrigger function in C# from Visual Studio Code like that:

![vscode1](https://mnie.github.com/img/23-09-2019BranchDeleter/vscodecreate1.png)


![vscode2](https://mnie.github.com/img/23-09-2019BranchDeleter/vscodecreate2.png)


![vscode3](https://mnie.github.com/img/23-09-2019BranchDeleter/vscodecreate3.png)


![vscode4](https://mnie.github.com/img/23-09-2019BranchDeleter/vscodecreate4.png)

Or from Visual Studio like that:

![vs1](https://mnie.github.com/img/23-09-2019BranchDeleter/vscreate1.png)


![vs2](https://mnie.github.com/img/23-09-2019BranchDeleter/vscreate2.png)


![vs3](https://mnie.github.com/img/23-09-2019BranchDeleter/vscreate3.png)

Then you have to change an extension of a project file from `.csproj` to `.fsproj` beyond that, we want to explicitly set the value of `FSharp.Core` dependency so we set it to `4.7.0`. Next thing is to delete a `.cs` file which is not needed here and adds a new `.fs` file to the project. Content of this file should look like this:

```fsharp
module Runner

    open Microsoft.Azure.WebJobs
    open Microsoft.Extensions.Logging

    [<FunctionName("BranchDeleter")>]
    let deleteStaleBranches ([<TimerTriggerAttribute("0 0 6 * * *")>]myTimer: TimerInfo, log: ILogger, context: Microsoft.Azure.WebJobs.ExecutionContext) =
        ()

```

We have a ready empty project, so we could go further, to some logic. Because our repositories are located on AzureDevops I used the following library to get information about projects/pull requests/branches etc.: `Microsoft.TeamFoundationServer.ExtendedClient` but if you like the idea it wouldn't be a problem to switch usage of this library to one that would suit your needs.

After the installation of the library, the first thing to do would be to download all repositories and get all branches for them.

```fsharp
module Configuration =

    type Config =
        {
            repository: string
            accessToken: string
            project: string
        }

module Deleter =

    open Microsoft.Extensions.Logging
    open System
    open Microsoft.TeamFoundation.SourceControl.WebApi
    open Microsoft.VisualStudio.Services.Client
    open Microsoft.VisualStudio.Services.Common
    open Microsoft.VisualStudio.Services.WebApi
    open System.Threading

    let private connection(config: Configuration.Config) = 
        let basicCred = new VssBasicCredential("", config.accessToken)
        let creds = new VssClientCredentials(basicCred)
        
        new VssConnection(new Uri(config.repository), creds)

    let run (config: Configuration.Config) (log) =
        let conn = connection(config)
        let client = conn.GetClient<GitHttpClient>()
        async {
            let! repositories = client.GetRepositoriesAsync(config.project, Nullable(false)) |> Async.AwaitTask
        } |> Async.StartAsTask
```

As you may see the first thing we do here is to create a connection to AzureDevops with all data needed. Then we want to get a git client from the already created connection object. Then in an async block, we could get all repositories via method `GetRepositoriesAsync`.

Right now we have all repositories, the next step would be to download all branches for these repositories. Branch information for a repository could be achieved by this method:

```fsharp
module Deleter
    ...
    let private getBranches (config: Configuration.Config) (client: GitHttpClient) (log: ILogger) (repo: GitRepository) =
        async {
            let! branches = client.GetRefsAsync(config.project, repo.Id, "", Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), "", null, CancellationToken.None) |> Async.AwaitTask
            return branches
        }
    ...
```

So as you may notice we used `GetRefsAsync` method to download information about all branches in a single repository. Next thing is to delete all branches in a repository. To delete a branch we have to create a `delete object` which should be created concretely:
- `OldObjectId` should be set to the actual `ObjectId` from ref;
- `NewObjectId` should be equal to `00000000`
- `Name` should be set to value from ref;
- `RepositoryId` should be equal to repository id.

To proceed with those requirements I wrote the following code:

```fsharp
module Creator

    open Microsoft.TeamFoundation.SourceControl.WebApi
    open System

    let toDeleteObj repoId (ref: GitRef) =
        let newRef = GitRefUpdate()
        newRef.OldObjectId <- ref.ObjectId
        newRef.NewObjectId <- "0000000000000000000000000000000000000000"
        newRef.Name <- ref.Name
        newRef.RepositoryId <- repoId
        
        newRef
```

I think there is no need to going into this code, as long as it simply creates an object with valid property values. But as long as we have or know how to create a ref to delete we could use a function named `UpdateRefsAsync` which would delete branches. The whole combined code to fetch branches and delete them at the end looks like this:

```fsharp
let private deleteBranchesFrom (config: Configuration.Config) (client: GitHttpClient) (log: ILogger) (repo: GitRepository) =
    let createRefToDelete = Creator.toDeleteObj repo.Id

    async {
        let! branches = client.GetRefsAsync(config.project, repo.Id, "", Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), "", null, CancellationToken.None) |> Async.AwaitTask

        let branchesToDelete = 
            |> Seq.map createRefToDelete
        
        return! client.UpdateRefsAsync(branchesToDelete, repo.Id, config.project, null, CancellationToken.None) |> Async.AwaitTask
    } |> Async.StartAsTask

```

We could finish here, but we have to remember that we have some requirements about the branches that should be deleted. The first requirement was to not delete branches from which we have open pull requests. So to download all of those pull requests and get branches from which they were created we run following code:

```fsharp
...
let! branchesWithPR = client.GetPullRequestsByProjectAsync(config.project, Creator.pRCriteria ()) |> Async.AwaitTask
...
```

This method simply downloads all branches specified by criteria, where criteria look like this:

```fsharp
let pRCriteria () =
    let searchCriteria = GitPullRequestSearchCriteria()
    searchCriteria.Status <- Nullable(PullRequestStatus.Active)

    searchCriteria
```

Beyond branches that have opened pull requests, we also want to filter those which were not older than one month or have concrete names. So code like this was written:

```fsharp
module Filter

    open Microsoft.TeamFoundation.SourceControl.WebApi
    open System
    let private baseBranchesToExclude = [ 
            "test"; "beta"; "release";
            "refs/heads/test"; "refs/heads/beta"; "refs/heads/release";
        ]

    let notDeletable (branches: GitRef seq) (branchesWithPR: GitPullRequest seq) =
        let monthAgo = DateTime.UtcNow.AddMonths (-1)
    
        branches
        |> Seq.filter (fun x ->
            if (baseBranchesToExclude |> Seq.filter (fun y -> y = x.Name) |> Seq.length > 0) then false
            elif (branchesWithPR |> Seq.exists (fun y -> y.SourceRefName = x.Name)) then false
            elif (x.Statuses |> Seq.map (fun y -> y.UpdatedDate) |> Seq.sortDescending |> Seq.head |> fun z -> z > monthAgo) then false
            else true
        )
```

At first, we set the month ago date, then for which branch that was passed to a method as a `GitRef seq` we:
- check if the branch is named the same as those branches we don't want to delete (test/beta/release);
- check if the branch resides on a list of branches that have opened pull requests;
- check if the branch was updated after a month ago.
Otherwise, we want to delete a branch.

To sum up, what we have right now, we have filtering, gathering information about pull requests and branches. So the combination of all of the above looks like this:

```fsharp
let private deleteBranchesFrom (config: Configuration.Config) (client: GitHttpClient) (log: ILogger) (repo: GitRepository) =
    let createRefToDelete = Creator.toDeleteObj repo.Id
    async {
        let! branchesWithPR = client.GetPullRequestsByProjectAsync(config.project, Creator.pRCriteria ()) |> Async.AwaitTask
        let! branches = client.GetRefsAsync(config.project, repo.Id, "", Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), Nullable<bool>(), "", null, CancellationToken.None) |> Async.AwaitTask

        let branchesToDelete = 
            Filter.notDeletable branches branchesWithPR
            |> Seq.map createRefToDelete
        
        match branchesToDelete with
        | toDelete when toDelete |> Seq.length > 0 ->
            let! _ = client.UpdateRefsAsync(toDelete, repo.Id, config.project, null, CancellationToken.None) |> Async.AwaitTask
            ()
        | _ -> ()
            
    } |> Async.StartAsTask
```

You may notice that we have to add here additional check if the list of branches we want to delete is not empty if it is we don't want to do any action.

Going further we connect this code with a code responsible for fetching repositories.

```fsharp
let run (config: Configuration.Config) (log) =
    let conn = connection(config)
    let client = conn.GetClient<GitHttpClient>()
    let deleteFunc = deleteBranchesFrom config client log
    async {
        let! repositories = client.GetRepositoriesAsync(config.project, Nullable(false)) |> Async.AwaitTask

        do! 
            repositories
            |> Seq.map (fun repo -> async { do! deleteFunc (repo) |> Async.AwaitTask })
            |> Async.Parallel
            |> Async.Ignore
    } |> Async.StartAsTask
```

We run this code in our time trigger method:

```fsharp
module Runner

    open Microsoft.Azure.WebJobs
    open Microsoft.Extensions.Logging
    open Microsoft.Extensions.Configuration
    open Configuration

    [<FunctionName("BranchDeleter")>]
    let deleteStaleBranches ([<TimerTriggerAttribute("0 0 6 * * *")>]myTimer: TimerInfo, log: ILogger, context: Microsoft.Azure.WebJobs.ExecutionContext) =
        let builder = new ConfigurationBuilder()
        let configuration = builder.SetBasePath(context.FunctionAppDirectory).AddJsonFile("settings.json", true, true).AddEnvironmentVariables().Build()
        let config = {
            repository = configuration.["AzureDevops:Repository"]
            project = configuration.["AzureDevops:Project"]
            accessToken = configuration.["AzureDevops:AccessToken"]
        }
        async {
            async { do! (Deleter.run config log) |> Async.AwaitTask } |> Async.RunSynchronously
        } |> Async.StartAsTask |> ignore
```

The last thing we have to do is a deployment to Azure. So we set-up an AzureFunction on AzureDevops:

![createfunction](https://mnie.github.com/img/23-09-2019BranchDeleter/funcazure.png)

We go to the project and click publish on it. Set all the information and click publish.
If we enable AppInsights we could also check if our function is working:

![branchdeleter](https://mnie.github.com/img/23-09-2019BranchDeleter/branchdeleter.png)


![branchdeleter2](https://mnie.github.com/img/23-09-2019BranchDeleter/branchdeleter2.png)

To summarize, thanks to the above code we could keep our codebase in terms of branches in a good shape, without any not relevant branches that reside in repositories for months or even years. Because someone forgot to check a checkbox to delete his branch after a merge of a pull request. You could found the full source code [here](https://github.com/MNie/BranchDeleter).

Thanks for reading :)

