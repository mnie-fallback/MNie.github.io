---
layout: post
title: Struggles when migrating from ES 6.8 to ES 7.5
---

Hi,

in an application that I develop from day to day, I encounter a problem with the migration of data from ES 6.8.2 to ES 7.5.1. The normal attitude in our application would be to run re-indexation, which fetches data from multiple sources and index them in a cluster. But because our application works 24/7 because of clients all around the globe. There were situations that re-indexation fails for some companies, for some reason. Because we want to have an option to react immediately in a situation like that, we decided to write an ad hoc migrator. Which takes data from the old cluster (6.8.2) and "move" them to the new one (7.5.2). Worth noting is also that the new cluster was located on a new server, not the same as the old one.

This isn't a hard task, but we have to keep in mind that there would be some breaking changes between major releases of ES. Because of that, we decided that the easiest solution would be to create an application that would consist of 2 services. The first one would be responsible for gathering data from old elastic -> mapping them to some DTO -> sending them to the second service. While the second service would receive them, map them, and push them to the new cluster. Sounds easy right? So let's look at the code.

The structure of a project looks as follows. We have 2 subfolders named `ES6` and `ES7` which were aligned to the services responsible for gathering/inserting data from/to clusters. We started from a service whose main task was to fetch data. We have here a `Fetcher` module:

```fsharp
module Fetcher =
  type Type =
    | Init
    | Next of string
    | Last
        
  let private client conString =
    let url = Uri(conString)
    let pool = new SingleNodeConnectionPool(url)
    let builder = new ConnectionSettings(pool)
    let con = builder
      .ThrowExceptions()
      .RequestTimeout(TimeSpan(0, 0, 30))
      .MaxRetryTimeout(TimeSpan(0, 0, 62))
      .MaximumRetries(2)
      .EnableHttpCompression()
      .EnableHttpPipelining(false)
      .PrettyJson()
      .EnableTcpKeepAlive(TimeSpan(0, 10, 0), TimeSpan(0, 0, 10))
    ElasticClient con
        
  let fetchFor conString companyId ``type`` =
    let resolveType (docs: IReadOnlyCollection<SomeModel>) scrollId =
      if docs.Count > 0 then
        Next scrollId
      else Last
        
    match ``type`` with
    | Init ->
      async {
        let! result = 
          (client conString)
            .SearchAsync<SomeModel>(
              fun (s:SearchDescriptor<SomeModel>) ->
                new SearchRequest(
                  Size = (Nullable<int> 1000),
                  From = (Nullable<int>0),
                  Scroll = Time(TimeSpan.FromMinutes(1.)),
                  Query = new QueryContainer(query = BoolQuery(Should = [
                    new QueryContainer(query = new TermQuery(
                      Field = new Field("companyId"), Value = companyId))
                    ])
                  )
                ) :> ISearchRequest) |> Async.AwaitTask
                
        let t = resolveType result.Documents result.ScrollId            
        return (t, result.Documents)    
      }
      | Next scrollId ->
        async {
          let! result =
            (client conString)
              .ScrollAsync<SomeModel>(Time(TimeSpan.FromMinutes(1.)), scrollId)
              |> Async.AwaitTask
                
          let t = resolveType result.Documents result.ScrollId            
          return (t, result.Documents)    
        }
        | Last -> async { return (Last, new List<SomeModel>() :> IReadOnlyCollection<SomeModel>) }       
```

As we can see above. In case of an initial request, we use a `SearchAsync` method, which would return n first records and a `scrollId` needed for gathering the next bunch of data.

When we have all the data, we check if we would need another call for data, or it is the last call for them.

If more data needs to be fetched we are using a `ScrollAsync` method to which we are passing previously gathered `scrollId`.

That's all for the `Fetcher` module. The next one is a module responsible for mapping data to DTO.

But we gonna skip code example here, since it's a normal mapping without any logic inside. The next `block` is a `Migration` module. 

```fsharp
let run (client: HttpClient) data =
  let serialized = JsonConvert.SerializeObject(data)
    async {
      let! result = client.PostAsync("http://someaddress/migration", new StringContent(serialized, Text.Encoding.UTF8, "application/json")) |> Async.AwaitTask
      if result.StatusCode = HttpStatusCode.OK then
        return Microsoft.FSharp.Core.Result.Ok ()
      else
        let! content = result.Content.ReadAsStringAsync() |> Async.AwaitTask
        return Microsoft.FSharp.Core.Result.Error (content)
    }
```

Which send mapped data via HTTP to the second service and return error information if there is some. The whole code is combined in an `Application` module. As you can see we don't combine all of the data related to a company, because we could get a couple of thousands/millions records and that could result in some performance issues.

Going to the ES7 service. The first thing we are doing is mapping from a DTO to our `domain` model in a `Mapping` module.

As previously, there is no logic here, so we gonna skip the borring stuff. Only assigning values to properties/fields. After we are done with the mapping we "send" them to the `Uploader` module. 

Which is responsible for doing a `bulk` operation on an ES cluster and based on a result from a bulk operation, return information to an ES6 service.

```fsharp
module Uploader =
  let private client conString =
    let url = Uri(conString)
    use pool = new SingleNodeConnectionPool(url)
    let builder = new ConnectionSettings(pool)
    let con = 
      builder
        .ThrowExceptions()
        .RequestTimeout(TimeSpan(0, 0, 30))
        .MaxRetryTimeout(TimeSpan(0, 0, 62))
        .MaximumRetries(2)
        .EnableHttpCompression()
        .EnableHttpPipelining(false)
        .PrettyJson()
        .EnableTcpKeepAlive(TimeSpan(0, 10, 0), TimeSpan(0, 0, 10))
        .DefaultIndex("dashboard")
        .EnableDebugMode()
    ElasticClient con
        
  let upload conString docs =
    let toUpload =
      docs
        |> Array.map (
          fun (d: ES7.Model.SomeModel) ->
            let b = BulkCreateOperation<ES7.Model.SomeModel>(d)
            b.Id <- Id(d.IndexId)
            b.RetriesOnConflict <- Nullable<int>(1)
            b :> IBulkOperation
        )
        |> fun x -> new BulkOperationsCollection<IBulkOperation>(x)
      let request = BulkRequest()
      request.Operations <- toUpload
      async {
        let! result = (client conString).BulkAsync(request) |> Async.AwaitTask
        if result.Errors then
          return Some (result.ItemsWithErrors |> Seq.map (fun x -> x.Error.Reason))
        else
          return None
      }
```

To sum up, in a simple way we could write an application that would transmit some data from one to another cluster is fairly time (a couple of seconds for companies with hundreds of thousands of records). Is this a solution I would recommend to anyone? No, there is a build-in snapshot option that would be perfect for such occasion, then we could simply reindex all "companies" only for missing data. Beyond of snapshot option there is also a built-in [reindex](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/reindex-upgrade-remote.html). Also, the reindex application should be stable and shouldn't fail. But thanks to that small bastard we were able to move data for 2 companies on a production environment in a couple of seconds because the reindex process fails for them ~3 times and a fix could take up to 10-15 minutes to release it.

Thanks for reading!