---
layout: post
title: Running Canopy in Docker container
---

Hi,

I struggle a little bit with running Canopy inside a docker container. So I thought it would be great to document how to achieve it in a separate post. I have following code which is responsible for downloading a whole page by going to it.

```fsharp
let fetchHtml (city) =
    startChrome ()
    
    url "http://www.licytacje.komornik.pl/Notice/Search"
    waitFor search
    ".city" << city
    click "#Type"
    click "Nieruchomość"
    click ".button_next_active"

    let page = browser.PageSource
    quit ()

    page
```

The `startChrome` method looks as follows:

```fsharp
let private search () =
    (elements ".city").Length > 0

let private startChrome () =
    let chromeOptions = ChromeOptions()
    chromeOptions.AddArgument("--no-sandbox")
    chromeOptions.AddArgument("--headless")
    let chromeNoSandbox = ChromeWithOptions(chromeOptions)
    start chromeNoSandbox
```

As you may see I don't use HeadlessChrome instead of it I use standard chrome and pass arguments to it to use a `headless` mode and `no-sandbox`. Which are needed.

```fsharp
module Auctions =
    let get =
        configuration.chromeDir <- AppDomain.CurrentDomain.BaseDirectory

        Fetcher.fetchHtml
```

That's all from a code perspective, but I need to do some adjustments so the canopy would be run inside of a docker. To make it happen I have to do the following things:

- copy chrome driver in a server.fsproj to output folder;
```fsharp
<ItemGroup>
<Content Include="chromedriver.exe">
    <CopyToOutputDirectory>CopyIfNewer</CopyToOutputDirectory>
</Content>
<Content Include="chromedriver">
    <CopyToOutputDirectory>CopyIfNewer</CopyToOutputDirectory>
</Content>
</ItemGroup>
```
- install chrome in a docker image.
```fsharp

ENV CHROME_DRIVER_VERSION 79.0.3945.36

RUN apt-get update && apt-get install -y gnupg2 && apt-get install -y wget

RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
      && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
      && apt-get update \
      && apt-get install xvfb unzip google-chrome-stable -y \
      && wget https://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip \
      && unzip -d /usr/local/bin chromedriver_linux64.zip
```

This article in short illustrates how to run canopy inside of docker container. You could check-out [my repository](https://github.com/MNie/CanopyInDocker) the get the fully working example of above.

Thanks!