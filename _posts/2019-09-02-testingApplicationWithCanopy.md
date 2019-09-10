---
layout: post
title: Testing application continuously with Canopy and Azure Pipelines
---

Hi,

in a side project in which I'm working right now, we encounter a problem that after we made some changes we break some other functionality on production. It's always a pain in a heart when you discover that you break up something on production and have to fix that immediately. Fix to production arrives in less than a minute, but I don't want to have such situations in a future, especially that I'm not a person that discovers this error. It is also a little bit disappointing that besides having a nice base of unit and integration tests something like that happened.

So going into details the problem was that we passed invalid data from a front-end side to backend app when changing a value in one of the dropdowns. So the result was a `500` status code from backend.

So to leverage errors like this in a future I decided to cover application with automated tests but I have to keep in mind that those tests on production shouldn't add anything in a system, cause any action related to adding some products, etc. triggers an event that sends an email/notification to every interested person (financial related project). So those tests should be a smoke test on production. But thanks to them we would have fast feedback that some core functionalities don't work correctly.

Because of business logic which is written in C# and F# automatic tests were written in F# thanks to a `Canopy` library. So I started from configuring a project to run tests on a chrome browser. So I created a project and add below NuGet packages to it:

```xml
<PackageReference Include="Canopy" Version="2.1.0" />
<PackageReference Include="Selenium.WebDriver.ChromeDriver" Version="2.45.0" />
```

The first test should be responsible for login to an application. It should enter the page, fill the box for login and password and then click sign-in. Then we should be redirected to the main application view. The test looks like this:

```fsharp
module LoginTests =
    let all () =
        "check if login works" &&& fun _ ->
            url "http://test.somecompany.com"
            
            "#login-email" << "user@domain.com"
            "#login-password" << "2137"
            click "SIGN IN"
            
            ".page-title" == "Dashboard"
            
            click "#UserDropdown"
            click "Logout"
            
            (read "SIGN IN") == "SIGN IN"
```

After that we wrote some tests that should simply open a subpage and check if it loads correctly, and those tests looked like this:

```fsharp
module Smoke =
    let all (envUrl: string) =
        let login (userName) =
            url envUrl
            click " Back to home"
            "#login-email" << userName
            "#login-password" << "hardPass"
            click "SIGN IN"
        
        let logout() = 
            click "#UserDropdown"
            click "Logout"
    
        let waitForTable () =
            (elements "tr").Length >= 2
    
        let action (userName, page) =
            login (userName)    
            click page
            
            waitFor waitForTable
                
            logout()
    
        "check Orders page for Standard User" &&& fun _ -> action ("customer@someCompany.com", "Orders")
        "check Messages page for Standard User" &&& fun _ -> action ("customer@someCompany.com", "Messages")
        
        "check Orders page for Creator" &&& fun _ -> action ("customerAnother@someCompany.com", "Orders")
        "check Messages page for Creator" &&& fun _ -> action ("customerAnother@someCompany.com", "Messages")
        
        "check Documents page for Admin" &&& fun _ -> action ("admin@someCompany.com", "Documents")
        "check Notes page for Admin" &&& fun _ -> action ("admin@someCompany.com", "Notes")
        "check Orders page for Admin" &&& fun _ -> action ("admin@someCompany.com", "Orders")
        "check Messages page for Admin" &&& fun _ -> action ("admin@someCompany.com", "Messages")
```

As we could see, the test is very smooth and doesn't need any additional information to describe it. The only interesting things are operators and functions used by canopy ([the whole list could be found here](https://lefthandedgoat.github.io/canopy/actions.html)):

```fsharp
<< // insert value
== // assertion
click // click element with name, class, id
```

So this is how the first test looks like. How we could run it? We could use normal browser or a headless mode. When running locally on my computer I run it in a normal browser so I could see if everything is cool. So the code responsible for running test(s) looks like this.

```fsharp
[<EntryPoint>]
let main argv =
    let env =
        match argv.[0] with
        | "test" -> Env.Test "http://test.somecompany.com"
        | "prd" -> Env.Production "http://somecompany.com"
        | _ -> Env.Test "http://test.somecompany.com"
    configuration.pageTimeout <- 10.
    configuration.compareTimeout <- 20.
    configuration.failFast := true
    configuration.reporter <- new LiveHtmlReporter(ChromeHeadless, configuration.chromeDir) :> IReporter
    
    start Chrome
    resize (1400, 900)
    
    let liveReporter = configuration.reporter :?> LiveHtmlReporter
    liveReporter.reportPath <- Some "reports/AutomationResults"
    
    let tests () =
        match env with 
        | Env.Test t -> 
            LoginTests.all ()
            Smoke.all (t)
            BusinessLogic.all ()
            BusinessLogic.all ()
        | Env.Production p ->
            LoginTests.all ()
            Smoke.all (p)
    
    tests ()
    run ()
    quit ()
    
    canopy.runner.classic.failedCount

```

At first, we choose a browser, then set some options to a browser like width, height, etc. Set a reporter so we could get a nice report after test run. Then we invoke action which contains our test(s). In the end, we want to close a browser and return information about failed tests.

So we don't have anything to do right now, we could run tests! But when I run them at first I get an error that chrome driver doesn't exist in a particular directory.

![driver](https://mnie.github.com/img/02-09-2019-canopytests/driver.png)

Right now we have two options we could ensure that the driver would be always available in a path, or we could do the same as I did so I link the driver to a project.
There are some up and downsides of this solution. I have the full control of driver which is used to run tests, but on the other hand, this driver could not be in a line with the installed browser on a machine or agent, especially if it is a predefined Azure agent.

![downsides](https://mnie.github.com/img/02-09-2019-canopytests/downsides.png)

But for me it was good enough, so I have to do one more thing, set a path to a driver in the main method like this:

```fsharp
configuration.chromeDir <- AppDomain.CurrentDomain.BaseDirectory
```

So when we run tests right now, everything should be cool. 

![result1](https://mnie.github.com/img/02-09-2019-canopytests/result1.png)

So the next step would be to configure them on CI. The expected result would be to run them after every deploy to test/prod environment in a build pipeline. So every predefined agent on azure DevOps should have a chrome/firefox browser preinstalled. But when we configure a step in our pipeline like this:

![pipeline](https://mnie.github.com/img/02-09-2019-canopytests/pipeline.png)

And we run this pipeline, we get an error like this. 

![result](https://mnie.github.com/img/02-09-2019-canopytests/result2.png)

This is because an agent is not able to run chrome in a normal mode. The solution to this problem is to use chrome in a headless mode. We adjust a code:

```fsharp
...
configuration.reporter <- new LiveHtmlReporter(ChromeHeadless, configuration.chromeDir) :> IReporter
    
start ChromeHeadless
...
```

We run pipeline one more time and tests are green/red depending on the current status of the environment ;).

![result3](https://mnie.github.com/img/02-09-2019-canopytests/result3.png)

To sum up, because of those small things we could easily write and run some base automated tests on azure DevOps after each deployment including a report of them. Which helps us in keeping the integrity and reliability of key functionalities in our app between front and backend layer.

Thanks for reading :)

