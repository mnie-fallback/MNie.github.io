---
layout: post
title: Testing GraphQL queries with FsCheck library
---

Hi,

some time ago I had a pleasure to join a project which uses a [GraphQL library](https://graphql.org/). GraphQL essentially allows users to define queries from a front-end to gather concrete data. So as it looks it is completely different than a standard REST endpoint. GraphQL differs queries which only gather data from those which also mutates them. In this article, I want to concentrate on queries which only gather some data and method to test them.

I don’t want to go deep into GraphQL details and how to create some GraphQL query. I assume that we have a query in which we could ask about car details. Model for beforehand said `Car` looks like this:

```csharp
public class Engine
{
    public string Indicator { get; set; }
    public int Power { get; set; }
    public int Displacement { get; set; }
}
public class Car
{
    public string Name { get; set; }
    public string Model { get;set; }
    public Engine Engine { get;set; }
}
```
GraphType:
```csharp
public class EngineGraphType : ObjectGraphType<Engine>
{
    public EngineGraphType(IEngineProvider engineProvider)
    {
        this.Name = "Engine";

        this.Field<NonNullGraphType<StringGraphType>>(
            "indicator", 
            resolve: ctx => ctx.Source.Indicator
        );
        this.Field<NonNullGraphType<IntGraphType>>(
            "power", 
            resolve: ctx => ctx.Source.Power
        );
        this.Field<NonNullGraphType<IntGraphType>>(
            "displacement", 
            resolve: ctx => ctx.Source.Displacement
        );
}

public class CarGraphType : ObjectGraphType<Car>
{
    public CarGraphType(IEngineProvider engineProvider)
    {
        this.Name = "Car";

        this.Field<NonNullGraphType<StringGraphType>>(
            "name", 
            resolve: ctx => ctx.Source.Name
        );
        this.Field<StringGraphType>(
            "model", 
            resolve: ctx => ctx.Source.Model
        );
        this.Field<EngineGraphType>(
            "engine",
            resolve: ctx => engineProvider.Provide(ctx.Source.Model)
        );
    }
}
```
And finally a query:
```csharp
public class CarQuery : RootQueryField
{
    private readonly ICarResolver _carResolver;

    public CarQuery(ICarResolver carResolver)
    {
        this._carResolver = carResolver;
        this.Name = "car";

        this.Arguments = new QueryArguments(
            new QueryArgument<NonNullGraphType<StringGraphType>> 
            { 
                Name = "name" 
            }
        );

        this.Resolver = new FuncFieldResolver<Task<Car>>(this.Resolve);

        this.Type = typeof(CarGraphType);
    }

    private Task<Car> Resolve(ResolveFieldContext ctx)
    {
        var param = ctx.Arguments.ToObject<ConcreteArguments>();
        return _carResolver
            .ResolveAsync(param);
    }
}
```

Thanks to that we could ask for a data a GraphQL endpoint like this:

```javascript
car(name: "name of a car")
{
    name
    model
    engine {
        indicator
        power
        displacement
    }
}
```

Of course, as I said before we don't need to specify all fields in an expected result. The query could be highly modifiable, so I thought it would be great idea to write some FsCheck tests, which would scan a GraphQL graph with a query and based on them generates sample queries. Thanks to which I could check a couple of properties. One of them would be to check if there are not any errors sections in GraphQL response, the second one would be to check if all responses end with a 200 HTTP Status Code. Thanks to the above I could be one hundred percent sure that all queries would work fine (maybe not from a business perspective :)), especially that those tests would be included in a CI pipeline as a part of integration tests.

To use a FsCheck library I have to write a generator for an input data. As an input data, I assume a serialized query. The project is written in a C#, so I used this language to write a generator.

Starting from a generic generator signature I assume that every query has to implement a method to collect data needed to create queries with arguments on their own. Also, every generator would know which class represents those arguments. So the signature of a generator looks as follows.

```csharp
abstract class QueryArb<TGraphQuery, TArguments>
```

As we see `TGraphQuery` is responsible for a query type which we want to be our `subject under test` while `TArguments` stands for an accepted arguments by query.
Going to an implementation of a generator I would present the whole code and then describe it line by line.

```csharp
public static Arbitrary<QueryTest> BuildArb(
    Func<IEnumerable<string>, Gen<TArguments>> fetchArgsFunc
)
{
    var container = ContainerFactory.Build();
    
    var query = container
        .Resolve<IEnumerable<RootQueryField>>()
        .FirstOrDefault(x => (x as TGraphQuery) != null);
    var queryType = ResolveQueryType(query.Type);
    var graphType = (IObjectGraphType)container
        .Resolve(queryType);

    var hasArguments = query.Arguments != null && query.Arguments.Any();
    var data = hasArguments
        ? GenWithArguments(fetchArgsFunc, query, graphType, container)
        : GenWithoutArguments(query, graphType, container);

    return data.ToArbitrary();
}
```

We start by building a container for all dependencies. Next, we gather query of concrete `IGraphType` type (our generic type) from all registered `RootQueryFields`, we have to remember that all queries are registered as `RootQueryFields` so if we would try to gather concrete `IGraphType` from an AutoFac container we would get an exception. Then we get information about the graph and its instance from a container cause we need an information about defined GraphQL fields for that query which are defined in a constructor. Also, we need to retain that those types could be also a generic type like `ListGraphType<OurCustomGraphType>` so we want to gather a generic argument. We could do that thanks to the below method:

```csharp
private static Type ResolveQueryType(Type queryType)
{
    var @type = queryType.IsGenericType 
        ? queryType.GenericTypeArguments.First() 
        : queryType;
    return @type.IsGenericType
        ? ResolveQueryType(@type)
        : @type;
}
```

As we can see I don't have here a stop guard cause I suppose (maybe incorrectly) that people don't create infinite generic types.
Next phase is a check if query accepts some arguments, if yes we want to create a query with them if nope we only need fields for which we could ask. We are going to the implementation of a generator with arguments.

```csharp
private static Gen<QueryTest> GenWithArguments(
    Func<IEnumerable<string>, Gen<TArguments>> fetchArgsFunc, 
    RootQueryField query, 
    IObjectGraphType graphType, 
    IContainer container
)
{
    var argNames = query.Arguments.Select(x => x.Name);
    var name = query.Name;
    var fields = graphType.Fields
        .Select(x => GetFieldRepresentation(x, container, 0));

    var fieldsSublist = Gen.SubListOf(fields);
    var arguments = fetchArguments(argNames);

    return
        from selectedFields in fieldsSublist
        from arg in arguments
        where selectedFields.Any()
        select BuildQuery(name, arg, selectedFields);
}
```
As we could see we get all arguments, then query name and all fields for which we could ask. Implementation of a `GetFieldRepresentation` looks as follows:

```csharp
private static string GetFieldRepresentation(
    IFieldType field, 
    IContainer container, 
    int deep
)
{
    if (deep > 3)
        return string.Empty;
    var customType = field.Type.IsGenericType 
        ? field.Type.GenericTypeArguments.First() 
        : field.Type;
    return container.IsRegistered(customType)
        ? GetFieldRepresentation(container, field, customType, deep)
        : field.Name;
}
```
It based on gathering field name if the type of this field is a simple GraphlQL type (for example IntGraphType etc. ) if the type is more complex we want to gather all fields of this complex field as long as we reach a max deep of 3.

Going back to the previous method. Based on gathered fields, which are in a form of string list, we create a generator rest on them. Then, we get all arguments for a query based on a function which is passed as an argument. Next, at the end of this function we create our final generator which depends on fields and arguments, we also keep in mind that our query has to contain some fields, so we have to filter out a situation where fields list is empty. 
Finally, we build test object thanks to the `BuildQuery` function, which looks like this:

<script src="https://gist.github.com/MNie/d8e8e1e5e3825766050c6f956eb35d53.js"></script>
```csharp

internal class QueryTest
{
    public readonly string Query;
    public QueryTest(string query) =>
        Query = query;
}

```
As it seems it isn't a rocket science, only a combination of a graphQL query. 

Implementation of a method responsible for the creation of a query without arguments looks very similar to this with arguments:

```csharp
private static Gen<CarTest> GenWithoutArguments(
    RootQueryField query, 
    IObjectGraphType graphType, 
    IContainer container
)
{
    var name = query.Name;
    var fields = graphType.Fields
        .Select(x => GetFieldRepresentation(x, container, 0));

    var fieldsSublist = Gen.SubListOf(fields);

    return
        from selectedFields in fieldsSublist
        where selectedFields.Any()
        select BuildQuery(name, EmptyArguments, selectedFields);
}
```
The question is how concrete generator for cars would look like?

```csharp
internal class CarArguments
{
    public readonly string name;

    public CarArguments(string name)
    {
        this.name = name;
    }
}

internal class CarArb : QueryArb<CarQuery, CarArguments>
{
    public static Arbitrary<QueryTest> Build() =>
        BuildArb(FetchArgs);

    private static Gen<CarArguments> FetchArgs(IEnumerable<string> argNames)
    {
        var data = new []{23, 45};
        return from single in Gen.Elements(data)
                select new CarArguments(single);
    }
}

```

As we could see generator for a concrete query is very simple and readable, tests look like this:

```csharp
//_client is a HttpClient from a Microsoft.AspNetCore.TestHost.TestServer
[FsCheck.NUnit.Property(Arbitrary = new[] { typeof(CarArb) }, MaxTest = 100)]
public Property Query_returns200(QueryTest query)
{
    var response = _client.GetAsync($"?query={query}").GetAwaiter().GetResult(); 
    //cause await seems to be not working fine with FsCheck 2.9 
    //(when method is marked as an async)

    var result = response.StatusCode;

    return result.Equals(HttpStatusCode.OK)
        .ToProperty()
        .Label($"{response.StatusCode} = 200");
}

[FsCheck.NUnit.Property(Arbitrary = new[] { typeof(CarArb) }, MaxTest = 100)]
public Property Query_HaveNoErrorsSection(QueryTest query)
{
    var response = _client.GetAsync($"?query={query}").GetAwaiter().GetResult(); 
    //cause await seems to be not working fine with FsCheck 2.9 
    //(when method is marked as an async)
    var content = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();

    var result = ErrorSections(parsedContent);
    return result.IsEmpty()
        .Equals(true)
        .ToProperty()
        .Label($"{result.Join("\r\n")} = empty array");
}

public static IReadOnlyCollection<string> ErrorSections(string content)
{
    var jObj = JObject.Parse(content);
    foreach (var pair in jObj)
        if (pair.Key.Equals("errors", StringComparison.OrdinalIgnoreCase))
            yield return pair.Value.ToString();

    return new string[0];
}
```
To sum up, we could create very easily a FsCheck generator so that a GraphQL query would be tested almost in 100% in case of some lacks in implementation. It helps us in delivering a new version of our application which couldn't be spotted while writing some single unit tests or manual tests. 

Thanks for reading!
