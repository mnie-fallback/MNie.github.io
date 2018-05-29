---
layout: post
title: Testing GraphQL queries with FsCheck library - Union GraphTypes
---

Hi,

Todays once more I want to focus on testing GraphQL queries via FsCheck library. This post is a continuation of previous two posts ([first](https://www.mnie.me/2018-05-07-graphQLTesting/), [second](https://www.mnie.me/2018-05-21-graphQLTestingPart2/)). Some time ago I changed one of the queries in our project, so it could accept as a value for one field value A or B. I decided to use for that an UnionGraphType which is available in GraphQL. Implementation of a car query and graph types from previous articles looks as follows.

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

Query:

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

In our case the use of an UnionGraphType we could be named as an additional option available in a car. To visualize it better, we could assume that car could have some media system which could be a CD radio, Android radio, Cassette radio. Above three classes we implement as following GraphTypes.

```csharp
public class CDRadio
{
    public bool discChanger { get; set; }
    public string Model { get;set; }
}

public class AndroidRadio
{
    public int systemVersion { get; set; }
    public string Model { get;set; }
}

public class CassetteRadio
{
    public string Model { get;set; }
}

public class CDRadioGraphType : ObjectGraphType<CDRadio>
{
    public CDRadioGraphType(IEngineProvider engineProvider)
    {
        this.Name = "CD";

        this.Field<BooleanGraphTyp>(
            "discChanger", 
            resolve: ctx => ctx.Source.DiscChanger
        );
        this.Field<StringGraphType>(
            "model", 
            resolve: ctx => ctx.Source.Model
        );
    }
}

public class AndroidRadioGraphType : ObjectGraphType<AndroidRadio>
{
    public AndroidRadioGraphType(IEngineProvider engineProvider)
    {
        this.Name = "Android";

        this.Field<IntGraphType>(
            "systemVersion", 
            resolve: ctx => ctx.Source.SystemVersion
        );

        this.Field<StringGraphType>(
            "model", 
            resolve: ctx => ctx.Source.Model
        );
    }
}

public class CassetteRadioGraphType : ObjectGraphType<CassetteRadio>
{
    public CassetteRadioGraphType(IEngineProvider engineProvider)
    {
        this.Name = "Cassette";

        this.Field<StringGraphType>(
            "model", 
            resolve: ctx => ctx.Source.Model
        );
    }
}
```

Definition of a UnionGraphType strapping all these types looks like this:

```csharp
public class MultimediaSystem : UnionGraphType
{
    public MultimediaSystem(
        CDRadioGraphType cd, 
        AndroidRadioGraphType android, 
        CassetteRadioGraphType cassette
    )
    {
        this.Type<CDRadioGraphType>();
        this.Type<AndroidRadioGraphType>();
        this.Type<CassetteRadioGraphType>();
    }
}
```

Right now our custom UnionGraphType is implemented, although we would like that our queries would be still testable by FsCheck code written in previous posts. So to do that, we have to make a couple of changes so that query would be generated correctly. Right now the full query with information about the multimedia system in a car looks like this:

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
    mediaSystem {
        __typename
        ... on CD {
            discChanger
            model
        }
        ... on Android {
            systemVersion
            model
        }
        ... on Cassette {
            model
        }
    }
}
```

As we could see a query for gathering fields specified as a union type are different than a query for a simple field. Instead of `field`, we have to write `... on field`. Because of that while scanning all available fields we have to extract those fields which are defined as a UnionGraphType. Important is that UnionGraphType doesn't inherit from IComplexGraphType, this is why test without any change would fail when trying to produce a query. So right now the first thing when scanning the available fields is to check if the field is a UnionGraphType or not. If it is we want to collect all available GraphTypes for this Union and for every of this type generate a part of a query but also keep the actual behavior for fields not defined as unions. Here is a piece of code which is responsible for scanning all the fields:

```csharp
private static string ToString(IContainer container, IFieldType field, Type toResolve, int deep) =>
    toResolve.IsAssignableTo<IComplexGraphType>()
        ? ToStringAsField(container, field, toResolve, deep, FieldFormat)
        : ToStringAsUnion(container, field, toResolve, deep);

private static string FormatWithPropertiesForUnions(IContainer container, IFieldType field, Type toResolve, int deep)
{
    var unionTypes = (UnionGraphType) container.Resolve(toResolve);
    var unions = unionTypes.Types
        .Select(type => FormatAsField(container, field, type, deep, UnionFormat));
    return string.Join("", unions);
}

private static string FormatAsField(
    IContainer container, 
    IFieldType field, 
    Type toResolve, 
    int deep, 
    Func<string, string, string, string> format
)
{
    var type = (IComplexGraphType) container.Resolve(toResolve);
    var fields = string.Join(' ', type.Fields.Select(y => ToString(y, container, deep 1)));
    return string.IsNullOrWhiteSpace(fields)
        ? string.Empty
        : format(field.Name, type.Name, fields);
}
 
private static string FieldFormat(string fieldName, string _, string fields) =>
    $"{fieldName}{{{fields}}}";

private static string UnionFormat(string fieldName, string unionName, string fields) =>
    $"{fieldName}{{ ... on {unionName}{{{fields}}}}}";

```

Thanks to the above fixes we were able to test also queries which contain a UnionGraphTypes as fields. The only dilemma here is that we ask in a query for all possible types specified in a union. The solution to that is to run some warmup or startup query which would infer based on a `__typename` field, what type is available. But in our situation, this problem (to ask for all possible types) is rather small, and a solution to this would be overwhelming, so we decide to keep it as simple as possible.

Thanks for reading!
