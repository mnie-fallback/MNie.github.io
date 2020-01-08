---
layout: post
title: Visualization of estates on a map via Fable
---

Hi,

From time to time I look at bailiff auctions. To check if there are some interesting properties for sale. If I want to check where they are located I have to go into details of each auction. I felt it would be great to see all of them on the map, so I could filter them by localization or district.

I decided to write an application that would gather information about auctions in FABLE. Based on that information I want to show markers on a map with a prize, date of auction and a link to full details of an auction inside of it.

I started from creating a blank project. (Thanks to [SAFE Template](https://github.com/SAFE-Stack/SAFE-template)):

```bash
dotnet new -i Bailif –server Saturn –communication remoting –deploy docker 
```

Thanks to the above I created a blank project. So I could start by writing the logic of an application. First of all, I want to download information about available properties auctions in the city. I decided that by default I would download data for `Gdańsk`. Looking at the site with auctions I see that communication is done by forms:

![forms](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/search.jpg)

The page gives a lot of possibilities to filter. In my situation only 2 of 26 options are relevant. City and the type of auction. Because of that, I look at how the request sent by a page looks like.

![formsfinal](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/searchfill.jpg)
![request](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/request.jpg)

Having information about how the request looks like I could implement it in code. I started by creating a new project named `Application` in my solution. In this project, all logic related to downloading/parsing data would be located. The page sends a simple POST request setting 2 of 26 fields, so the rewrite of this in F# looks as follow:

```fsharp
let fetchHtml (city) =
    let request =
        Request.createUrl Post "http://www.licytacje.komornik.pl/Notice/Search"
        |> Request.setHeader (Accept "application/json")
        |> Request.body (BodyForm 
            [
                NameValue ("Type", "1")
                NameValue ("City", city)
            ])
        |> Request.responseAsString
        |> run
    request
```

I used [http.fs](https://github.com/haf/Http.fs) and [hopac](https://github.com/Hopac/Hopac) libraries. In response, I get the full HTML page which I have to parse and gather the data I want to show on a map.

Going to parsing. I was thinking about using HtmlDocument, HtmlProvider, and HtmlAgilityPack. Because an app was written in F# the last option goes away. In terms of optimization of source files (I don't want to have files of hundred of lines, because of templates `send` to HtmlProvider) the second option was also refused. So I used HtmlDocument. List of auctions looks like that:

![list](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/list.jpg)

On a map, I want to show the localization of a property that would be auctioned. This information would be visualized as a marker on a map. The marker would have additional info with the data described earlier. Because of that for each record on a list, I have to gather the:
- price;
- date of the auction;
- link to details.
So I wrote a code like this:

```fsharp
let private onlyNumbers = Regex("[^0-9,]+", RegexOptions.Compiled)
let private baseUrl = "http://www.licytacje.komornik.pl"

let parseHtml (html) =
    let doc = HtmlDocument.Parse html
    
    doc.CssSelect("table")
    |> List.tryHead
    |> fun tab ->
        match tab with
        | Some x ->
            x.Elements [ "tBody" ]
            |> List.tryHead
            |> fun body ->
                match body with
                | Some v ->
                    v.Elements [ "tr" ]
                    |> List.skip 1
                    |> List.filter (fun y -> 
                        let nOfElements = 
                            y.Elements [ "td" ] 
                            |> List.length
                        nOfElements > 1
                    )
                    |> List.map (fun x ->
                        ...
                    )
                | _ -> []                                        
        | _ -> []
```

As you may notice. First, I parse the document. Look for a first table on a page and check if it has more than 1 record except for the header record. Otherwise, we want to pass an empty list. 
If the data seems to be correct, I gather the concrete information. Price and the details link are located in the last 2 columns of a table. So the following fold/reduce line was written to gather only those two pieces of information.

```fsharp
let (_, details) = 
    Seq.foldBack (fun e (i, acc) -> 
        (i - 1, 
            if i <= 0 then 
                acc 
            else 
                e :: acc)) columns (2, [])
let prize = 
    details 
    |> Seq.head 
    |> fun x -> x.InnerText () 
    |> fun x -> onlyNumbers.Replace(x, "")
let link = 
    details 
    |> Seq.last 
    |> fun x -> x.Elements [ "a" ] 
    |> Seq.head 
    |> fun x -> x.Attribute "href" 
    |> fun x -> x.Value
```

Beyond that, I also get the information about the date of the auction. So here is the full code for parsing list properties:

```fsharp
let private onlyNumbers = Regex("[^0-9,]+", RegexOptions.Compiled)
let private baseUrl = "http://www.licytacje.komornik.pl"

let parseHtml (html) =
    let doc = HtmlDocument.Parse html
    
    doc.CssSelect("table")
    |> List.tryHead
    |> fun tab ->
        match tab with
        | Some x ->
            x.Elements [ "tBody" ]
            |> List.tryHead
            |> fun body ->
                match body with
                | Some v ->
                    v.Elements [ "tr" ]
                    |> List.skip 1
                    |> List.filter (fun y -> 
                        let nOfElements = 
                            y.Elements [ "td" ] 
                            |> List.length
                        nOfElements > 1
                    )
                    |> List.map (fun x ->
                        let columns = x.Elements [ "td" ]
                        let ``when`` = 
                            columns.[2] 
                            |> fun x -> 
                                DateTime.ParseExact(x.InnerText().Trim(), "dd.MM.yyyy", CultureInfo.InvariantCulture)
                        let (_, details) = 
                            Seq.foldBack (fun e (i, acc) -> 
                                (i - 1, 
                                    if i <= 0 then 
                                        acc 
                                    else 
                                        e :: acc)) columns (2, [])
                        let prize = 
                            details 
                            |> Seq.head 
                            |> fun x -> x.InnerText () 
                            |> fun x -> onlyNumbers.Replace(x, "")
                        let link = 
                            details 
                            |> Seq.last 
                            |> fun x -> x.Elements [ "a" ] 
                            |> Seq.head 
                            |> fun x -> x.Attribute "href" 
                            |> fun x -> x.Value
                        {
                            prize = (Decimal.Parse(prize.Replace(",", "."), CultureInfo.InvariantCulture))
                            link = (new System.Uri(sprintf "%s%s" baseUrl (link ())))
                            ``when`` = ``when``
                        }
                    )
                | _ -> []                                        
        | _ -> []
```

Right now I have the full list of auctions. The only thing I still need is a localization of a home. I could get it from details of an auction, which looks like that:

![details](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/details.jpg)

So right now for each sale, I download the full detail page. This fetch would be much easier than the previous one. Because I could use a simple GET request, which gonna looks like that:

```fsharp
let fetchAuction link =
    let request =
        Request.createUrl Get link
        |> Request.setHeader (Accept "application/json")
        |> Request.responseAsString
        |> run
    request 
```

After the download, I have to gather information about the localization which is located under the attribute `hidden_address` of an input. In addition to that, I also download a description of an auction. The code that realizes that:

```fsharp
let parseAuction (html) =
    let doc = HtmlDocument.Parse html
    
    let address =
        doc.CssSelect("input#hidden_address")
        |> List.head
        |> fun x -> x.Attribute "value"
        |> fun x -> x.Value ()
    let description = 
        doc.CssSelect("div#Preview") 
        |> List.head 
        |> fun x -> x.InnerText ()
    async {
        let! coords = Coordinates.translateAddressToCoords address
        match coords with
        | Some c -> 
            return Some {
                description = description
                address = address
                point = c
            }
        | _ -> return None
    }

```

The address is in the following format `city, street, house number` because I want to show it on a map I need to somehow translate it to longitude and latitude. This is why I pass the `human-readable` address to a `translateAddressToCoord` function which has a task of reverse geocoding. To achieve it I used `nominatim` service to which I send a GET request and then parse a response to get lng/lat.

```fsharp
type CoordinatesResponse = JsonProvider<"""
[
  {
    "place_id": 101573834,
    "licence": "Data © OpenStreetMap contributors, ODbL 1.0. https://osm.org/copyright",
    "osm_type": "way",
    "osm_id": 111621751,
    "boundingbox": [
      "54.3054945",
      "54.3057362",
      "18.5823766",
      "18.5826803"
    ],
    "lat": "54.30561535",
    "lon": "18.58252845",
    "display_name": "Some Street 4, Gdańsk, województwo pomorskie, 80-180, Polska",
    "class": "building",
    "type": "yes",
    "importance": 0.5309999999999999
  }
]
""">

type Coordinate = decimal

type Point =
    {
        longitude: Coordinate
        latitude: Coordinate
    }

module Coordinates =
    let internal parseResponse (res: string) =
        CoordinatesResponse.Parse res
        |> Array.tryHead
        |> fun x ->
            match x with
            | Some v -> Some { longitude = v.Lon; latitude = v.Lat }
            | _ -> None
    
    let translateAddressToCoords (address) =
        async {
            let request =
                Request.createUrl Get (sprintf "https://nominatim.openstreetmap.org/search?q=%s&format=json" address)
                |> Request.setHeader (Accept "application/json")
                |> Request.setHeader (UserAgent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36")
                |> Request.responseAsString
                |> run
            do! Task.Delay (1000) |> Async.AwaitTask
            return request |> parseResponse
        }
```

Here I decided to use JsonProvider, because the response from service has a strict format. After sending a request, I parse it via JsonProvider and then gather longitude/latitude information.

Right now I have all the data needed. So the only thing to do is to combine them:

```fsharp
type internal Auction =
    {
        prize: Prize
        ``when``: Date
        link: Link
        details: Details
    }
    
type AuctionInformation =
    {
        prize: Prize
        link: Link
        description: Info
        ``when``: Date
        point: Point
    }

module Auctions =
    let private getConcreteAuction (auction: BaseAuction) =
        Fetcher.fetchAuction auction.link.AbsoluteUri
        |> Parser.parseAuction

    let get =
        configuration.chromeDir <- AppDomain.CurrentDomain.BaseDirectory

        Fetcher.fetchHtml
        >> Parser.parseHtml
        >> fun x ->
            let concreteAuctions = Fetcher.fetchAuctions (x |> List.map (fun y -> y.link.AbsoluteUri))
            async {
                let! result =
                    x
                    |> List.map (fun y ->
                        async {
                            let! details = getConcreteAuction (y)
                            match details with
                            | Some d ->
                                let info = 
                                    sprintf "Prize: %M zł, property located near: %s, auction at: %s" y.prize d.address (y.``when``.ToString("dd/MM/yyyy"))
                                return [{
                                    prize = y.prize
                                    link = y.link
                                    description = info
                                    point = d.point
                                    ``when`` = y.``when``
                                }]
                            | None -> return []
                        }
                    )
                    |> List.toSeq
                    |> Async.Parallel
                return result |> Seq.collect id
            }
```

and return as a model which is defined like that:

```fsharp
type Prize = decimal
type Link = string
type Info = string
type Address = string
type Date = System.DateTime
type Coordinate = decimal

type Localization =
    {
        longitude: Coordinate
        latitude: Coordinate
    }

type Auction =
    {
        prize: Prize
        link: Link
        description: Info
        ``when``: Date
        localization: Localization
    }
```

Mapping looks like that:

```fsharp
let mapToContract (ai: AuctionInformation): Shared.Auction =
    {
        prize = ai.prize
        link = ai.link.AbsoluteUri
        description = ai.description
        ``when`` = ai.``when``.Date
        localization = {
            longitude = ai.point.longitude
            latitude = ai.point.latitude
        }    
    }
```

I can go right now to the definition of the interface between the back and frontend. The interface would look like that:

```fsharp
type IAuctionsApi =
    { 
        init : unit -> Async<Result<Auction seq, exn>>
        filtered : string -> Async<Result<Auction seq, exn>>
    }
```

And implementation:

```fsharp
let auctionsApi = {
    init = fun () -> async {
        try
            let! auctions = Auctions.get ("Gdańsk")
            let mapped = 
                auctions 
                |> Seq.map mapToContract
            return Ok mapped            
        with
        | ex -> return Error ex          
    }

    filtered = fun (city) -> async {
        try
            let! auctions =
                match city with
                | c when String.isEmpty c -> Auctions.get ("Gdańsk")
                | _ -> Auctions.get (city)
            return auctions
                |> Seq.map mapToContract
                |> Ok
        with
        | ex -> return Error ex
    }
}
```

It would give a possibility to download the default data, and filtered one based on some keyword (city name).

As long as a shared model between client and server is done right now. The server is also ready. We could go to the frontend part of an application. Which needs to be adjusted. I start by showing a blank map. So I have to install 2 libraries:

```fsharp
Fable.Leaflet
Fable.ReactLeaflet
```

Next thing is to adjust messages that would be sent across the application. I created the following messages:

```fsharp
type Msg =
    | Search of string
    | SearchChanged of string
    | Filtered of Result<Auction seq, exn>
    | Init of Result<Auction seq, exn>
    | Error of exn
```

I thought that every user could:
- change input (SearchChanged);
- submit input (Search).
And the server could respond with:
- initial data (Init);
- filtered data (Filtered);
- error (Error).
As long as messages were defined as follow we could go to the message handling.

```fsharp
let update (msg : Msg) (currentModel : Model) : Model * Cmd<Msg> =
    match currentModel.Auctions, msg with
    | _, Search city ->
        let search = Cmd.ofAsync search (city) Filtered Error
        currentModel, search
    | _, SearchChanged city ->
        let nextModel = { currentModel with Search = city }
        nextModel, Cmd.none
    | _, Filtered possibleAuctions ->
        match possibleAuctions with
        | Ok auc ->
          let zoomToFirst = 
            auc 
            |> Seq.tryHead 
            |> fun x -> 
              match x with 
              | Some a -> 
                (Fable.Core.U3.Case3 (
                    a.localization.latitude 
                    |> float, 
                    a.localization.longitude 
                    |> float)
                ) 
              | None -> currentModel.Zoom  
          let nextModel = { currentModel with Auctions = auc; Zoom = zoomToFirst }
          nextModel, Cmd.none
        | Result.Error e -> currentModel, Cmd.ofMsg (Error e)
    | _, Init possibleAuction ->
        match possibleAuction with
        | Ok auc ->
          let nextModel = { currentModel with Auctions = auc }
          nextModel, Cmd.none
        | Result.Error e -> currentModel, Cmd.ofMsg (Error e)        
    | _, Error e -> 
        JS.console.log(sprintf "%s%s%s" e.Message " " e.StackTrace)
        currentModel, Cmd.none     
```

Based on a type of a message I change the actual state of an application (SearchChanged, Error, Init, Filtered) or ask the backend side for data (Search). Also if the message would have an error msg inside I send another command with `Error`.

The code looks pretty simple. You could notice that we have here a `cascade` of messages. For example, when a user submits an input a server action is invoked. When it returns some data, a `filtered` or `error` message to handle that are sent. One handling looks a little bit different. It is a `filtered` message handling, where the calculation of localization of the first home also happens. So the map could be moved to this point. No matter which city user would search. The map will auto adjust to a valid region. Of course, I could count here a centroid of all points but I want to keep it very simple.

The definition and handling of messages are done. So I could show a map without any markers. I would show a leaflet map. To add it, I wrote the following code:

```fsharp
module RL = ReactLeaflet

...

RL.map [ 
    RL.MapProps.Zoom 10.
    RL.MapProps.Style 
        [
            Height 500
            MinWidth 200
            Width Column.IsFull 
        ]
        RL.MapProps.Center model.Zoom 
    ] 
            (mapElements model)
```

I define zoom, width, height, the center of a map. There are some key aspects, that you couldn't omit if you want to show a leaflet map:
- you have to set some width/height of a map, otherwise, the map wouldn't show;
- you have to set map center, otherwise, the map would be grey;
- you have to add a tile layer;

```fsharp
ReactLeaflet.tileLayer 
    [ ReactLeaflet.TileLayerProps.Url "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
      ReactLeaflet.TileLayerProps.Attribution "&amp;copy <a href=&quot;http://osm.org/copyright&quot;>OpenStreetMap</a> contributors" ] 
    []
```
- you have to import leaflet styles;
```fsharp
importAll "../../node_modules/leaflet/dist/leaflet.css"
```
- you have to change the default imagepath for an icon, otherwise you would get errors in console;
```fsharp
Leaflet.icon?Default?imagePath <- "//cdnjs.cloudflare.com/ajax/libs/leaflet/1.3.1/images/"
```
- you should not forget to add to package.json leaflet packages
```javascript
{
    "leaflet": "1.6.0",
    "react-leaflet": "2.6.1"
}
```

Remembering above, a map should be visible.

![map](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/map.jpg)

Having a blank map, I could show some markers on it. Previously I show how to handle messages with data information about auctions. These auctions are available in an app state. Thanks to that, I could show some markers. This is why `mapElements` function was created:

```fsharp
RL.map 
    [ 
        RL.MapProps.Zoom 10.
        RL.MapProps.Style 
        [
            Height 500
            MinWidth 200
            Width Column.IsFull 
        ]
        RL.MapProps.Center model.Zoom
    ] 
           (mapElements model)
```

```fsharp
module RL = ReactLeaflet

...

let buildMarker (auction: Auction): ReactElement =
    RL.marker 
      [ 
        RL.MarkerProps.Position (Fable.Core.U3.Case3 (auction.localization.latitude |> float, auction.localization.longitude |> float)) ] 
      [ RL.popup 
          [ RL.PopupProps.Key (auction.link |> string)]
          [ Control.p 
              [] 
              [ label [] [ !!auction.description ] ]
            Control.p 
                [] 
                [ Button.a
                    [ Button.Size IsSmall
                      Button.Props [ Href (auction.link |> string) ] ]
                    [ Icon.icon [ ]
                        [ Fa.i [Fa.Brand.Github; Fa.FixedWidth] [] ]
                      span [ ] [ str "Go to bailif description" ] ] ] ] ]    

let mapElements model =
  let markers = model.Auctions |> Seq.map buildMarker |> Seq.toList
  tile :: markers
```

For each element in a table I get from a server I create a marker. Set the position of it and create a popup with a short description of the auction (price, date, link to details).

The map and markers are visible. The only missing thing is an input that would accept strings and would trigger a search "function" after pushing the `submit` button. Code that is responsible for rendering a button:

```fsharp
Box.box' [ ]
        [ Field.div [ Field.IsGrouped ]
            [ Control.p [ Control.IsExpanded ]
                [ Input.text
                    [ Input.Disabled false
                      Input.Value model.Search
                      Input.OnChange (fun e -> dispatch (SearchChanged e.Value)) ] ]
              Control.p [ ]
                [ Button.a
                    [ Button.Color IsPrimary
                      Button.OnClick (fun _ -> dispatch (Search model.Search)) ]
                    [ str "Search" ] ] ] ]
```

Handler of a Search message (just a reminder):

```fsharp
...
| _, Search city ->
    let search = Cmd.ofAsync search (city) Filtered Error
    currentModel, search
...
```

The whole application is ready so I create a docker image:

```fsharp
fake build --target docker
```

Small adjustment to build.fsx, so my image would have an additional tag.

```fsharp
let dockerFullName = sprintf "%s/%s" dockerUser dockerImageName

Target.create "Docker" (fun _ ->
    buildDocker dockerFullName
)
```

Push to docker hub:

```bash
docker push user/image:tag
```

Creation of Web App on Azure

![creation](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/dockersetup.png)

When the deployment is ready I need to do one additional thing (as SAFE stack [stated](https://safe-stack.github.io/docs/template-docker/)). We need to map port 8085 which is used by Giraffe to port 80.

```bash
az webapp config appsettings set --resource-group RESOURCE --name APPNAME --settings WEBSITES_PORT=8085
```

How finally the applications looks like:

![final](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/map.jpg)

Right now I could finish this article. But as you may see in the above picture there is a map without any markers. Why? Because a page from which I scrap the data change a little bit from time when I implement the whole application. In every request, there should be included a `_requestValidation` field and a `cookie`. I decided that instead of that I would use the `canopy` to gather a full page after some on-site filtering. I modified code responsible for downloading a list of auctions:

```fsharp
let private search () =
    (elements ".city").Length > 0

let private startChrome () =
    let chromeOptions = ChromeOptions()
    chromeOptions.AddArgument("--no-sandbox")
    chromeOptions.AddArgument("--headless")
    let chromeNoSandbox = ChromeWithOptions(chromeOptions)
    start chromeNoSandbox

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

As I said I used a `canopy` library. I open a headless chrome browser which is hidden for a client. I open a page, search for a city and click "search". In the end, I download the full page and that's all. Parsing would look the same, no changes required.

The code for downloading a single auction was also modified to use a `canopy`. I thought that it would be a huge benefit in terms of performance to open a browser once and for each auction go to the site and download it. Downloading all auctions at once is achieved like that:

```fsharp
let private startChrome () =
    let chromeOptions = ChromeOptions()
    chromeOptions.AddArgument("--no-sandbox")
    chromeOptions.AddArgument("--headless")
    let chromeNoSandbox = ChromeWithOptions(chromeOptions)
    start chromeNoSandbox

let fetchAuctions links =
    startChrome ()

    let rec fetchDetails alreadyFetched toCheck =
        match toCheck with
        | [] -> alreadyFetched
        | head::tail ->
            url head
            let page = browser.PageSource
            let v = (head, page)
            fetchDetails (v::alreadyFetched) tail

    let result = fetchDetails [] links        
    quit ()

    dict result
```

And invoked like that:

```fsharp
module Auctions =
    let get =
        configuration.chromeDir <- AppDomain.CurrentDomain.BaseDirectory

        Fetcher.fetchHtml
        >> Parser.parseHtml
        >> fun x ->
            let concreteAuctions = Fetcher.fetchAuctions (x |> List.map (fun y -> y.link.AbsoluteUri))
            async {
                let! result =
                    x
                    |> List.map (fun y ->
                        async {
                            let auction = concreteAuctions.[ y.link.AbsoluteUri ]
                            let! details = auction |> Parser.parseAuction
                            match details with
                            | Some d ->
                                let info = sprintf "Prize: %M zł, property located near: %s, auction at: %s" y.prize d.address (y.``when``.ToString("dd/MM/yyyy"))
                                return [{
                                    prize = y.prize
                                    link = y.link
                                    description = info
                                    point = d.point
                                    ``when`` = y.``when``
                                }]
                            | None -> return []
                        }
                    )
                    |> List.toSeq
                    |> Async.Parallel
                return result |> Seq.collect id
            }
```

And finally application looks like that:
![final](https://mnie.github.com/img/08-01-2020AuctionsVisualizationWithFable/marker.jpg)

The only minus of a `new` solution is the performance. Because the previous solution was a lot faster.

Right now everything should work just fine locally. I need some adjustments so the canopy would be run inside of a docker. To make it happen I have to do the following things:
- copy chrome driver in a server.fsproj to output folder;
- install chrome on a docker image.

To summarize. In this article, I show how to combine two possibilities of scraping data from a webpage (if it doesn't have an API). And also how to write a simple application that simply does something in Fable. I hope you enjoyed this article :)

Thanks!
Full repo [here](https://github.com/MNie/Bailif.Auctions)