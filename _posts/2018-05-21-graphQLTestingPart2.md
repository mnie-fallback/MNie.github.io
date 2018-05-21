---
layout: post
title: Testing GraphQL queries with FsCheck library - Shrinking input
---

Hi,

Today I want to continue a topic about testing graphQL queries thanks to the FsCheck library. How we could achieve that I describe briefly [here](https://www.mnie.me/2018-05-07-graphQLTesting/).
So I want to focus on a situation when our tests spot some problems. By default, we should get in a unit test output window information about the query which caused a failed test thanks to that we could copy-paste this query and check manually what is wrong.

But what would be when our query or graphTypes are very complex and not every field is incorrect? In a situation like this we would like to get the minimum query which produced a failure, so we could leverage time needed to spend on a manual check. We could achieve that by writing a custom shrinker. If some test fails shrinker tries to change somehow an input for next iterations of a test so we could get the minimum failure query in our example.

So to see this in some example, previously we write tests for a Query and GraphType which are defined as follows:

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

We could assume right now that `model` field is resolved incorrectly. This bug was spotted by our FsCheck tests, but because we don't implement a shrinker, query for which our test fails looks like this:

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

What we want is to get a minimum query which caused tests to be red, which should look like this:

```javascript
{
car(name: "name of a car") { model }
}
```

To implement shrinker in our case we have to modify our test input type. Previously it looks like this:

```csharp
internal class QueryTest
{
    public readonly string Query;
    public QueryTest(string query) =>
        Query = query;
}
```

Now we want this type to contains information about name, arguments, and fields and how to build the whole query. So it would look like this:

<script src="https://gist.github.com/MNie/37f4f9e9215185e92cd5fb6592c2897b.js"></script>

We modified an input test object, right now we have to go to the BuildArb function, which was responsible for building an Arb object. We have to modify it so at the end of a function based on a Gen object it would also accept a shrinker function which would be responsible for minimizing an input data.

```csharp
public static Arbitrary<QueryTest> BuildArb(
    Func<IEnumerable<string>, Gen<TArguments>> fetchArgsFunc
)
{
    ...
    var gen = hasArguments
        ? GenWithArguments(fetchArgsFunc, query, graphType, container)
        : GenWithoutArguments(query, graphType, container);

    return Arb.From(gen, Shrink);
}
```


As we could see instead of previously used `gen.ToArbitrary`, right now we use an `Arb.From` function which also accepts a shrinker parameter. How does shrinker function look like?

```csharp
public static IEnumerable<QueryTest> Shrink(QueryTest qt)
{
    var count = qt.Fields.Count();
    if (count == 1)
        yield break;
    for(var i = 0; i < count; ++i)
        yield return new 
            QueryTest(
                qt.Name, 
                qt.Arguments, 
                qt.Fields.Where((_, index) => index != i)
            );
}
```
 
Implementation of a shrinker is pretty straightforward. The first thing that comes to mind is a return type which is an IEnumerable of QueryTest instead of simply QueryTest.  This is because we create k combinations of an input against which our test would be run in n shrink phases. If we go deeply into implementation we could see that we want to generate some "shrunk examples" till the query would contain a single field (graphQL expect at least one field in a query). Then we go through all fields and we remove one of them. I think this example would explain it. As we remember the field `model` in a `car` was resolved incorrectly. FsCheck produces following test case in the first phase of a test (shrink = 0). The test was run against the following query.

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

In the next phase (shrink = 1) tests were run against:

```javascript
car(name: "name of a car")
{
    model
    engine {
        indicator
        power
        displacement
    }
}
```

```javascript
car(name: "name of a car")
{
    name
    engine {
        indicator
        power
        displacement
    }
}
```

```javascript
car(name: "name of a car")
{
    name
    model
}
```

next phase (shrink = 2):

```javascript
car(name: "name of a car")
{
    engine {
        indicator
        power
        displacement
    }
}
```

```javascript
car(name: "name of a car")
{
    model
}
```

```javascript
car(name: "name of a car")
{
    engine {
        indicator
        power
        displacement
    }
}
```

```javascript
car(name: "name of a car")
{
    name
}
```

```javascript
car(name: "name of a car")
{
    model
}
```

```javascript
car(name: "name of a car")
{
    name
}
```

As we could see after two phases (shrink = 2) we could gather a minimum query which produces failure:

```javascript
car(name: "name of a car")
{
    model
}
```

![result](https://mnie.github.com/img/21-05-2018-GraphQLShrink/result.png)

Of course, a solution like that has some drawbacks.  For example what with very complex fields? With such minimal implementation of a shrinker, we could get a query with a single field, but this field could have a lot of fields for example:

```javascript
car(name: "name of a car") { a { b { c { d { e { … } } } } } }
```

Although in our case such simple implementation is enough and it helps gather an information what exactly is wrong and also leverage the time needed to spend on manual checking of a query.

Thanks for reading! :)
