---
layout: post
title: GraphQL error propagation
---

Hi,

today I want to focus on available options to an error propagation while executing customers query. The GraphQL response mainly contains two sections, data, and errors. 

```javascript
{
    data: {},
    errors: {}
}
```

Inside of the data, we could find a data for which we were asking aside in the error section we could find information about all the errors which were returned while resolving fields.
 
While using a GraphQL-Dotnet library, when an exception occurs during the resolving phase of some fields a corresponding generic information would be included in the errors section. The question is what if we would prefer to somehow control this message? In such scenario, we should include error information of type ExecutionError in an ExecutionErrors table which is available as a field in a Context which is a part of every GraphQL request. What if we want to add error information somewhere deeper in our codebase? We could pass this ExeuctionErrors table as a parameter to the invoked methods and modify this array in these functions, this could be a little bit harmful to our application cause we have to deal with implicit resize calls on this array and also our code would be hardly coupled and if in the future there would be some breaking change in a library we would have to change almost every method in our codebase. 

So as you can see this solution is not in line with the SOLID principle or Clean Code. This is why I decided to design a solution thanks to which our codebase would be clean and the information about how to create an error and add it to a table would be done in a single place.

Keeping in mind that every Query and Mutation inherit from a FieldType I created an abstract class which would contain all information about how to create/add error to the ExecutionErrors table. This class looks as follows:

```csharp
internal abstract class RootField : FieldType
{
    protected async Task<TType> Resolve<TType>(ResolveFieldContext context, Func<ResolveFieldContext, Task<Result<TType>>> resolve)
    {
        var result = await resolve(context);
        if (result.IsSuccess)
            return result.Payload;

        var error = new ExecutionError(result.Error.Message);
        error.Data.Add("errorCodes", result.Error);
        context.Errors.Add(error);
        return default(TType);
    }
}
```

I thought that this abstract class has to implement a generic function (named `Resolve`) which main responsibility would be to resolve a field and check if some errors occur while resolving if yes it should add those errors to the `Data` field of the error section.

As you can see I used here a [ResultType](https://github.com/MNie/ResultType) which implementation you could find [here](https://github.com/MNie/ResultType) thanks to that I could distinguish if everything is cool or not. If `IsSuccess` is set to true we want to return a `Payload` of a `ResultType` which would be a plain data for currently resolved field, otherwise, if `IsFailure` is set to true we want to add information about errors to the `errorCodes` section.

Usage of this method in a resolver could look like this:

```csharp
internal class SomeQuery: RootField
{
    private readonly SomeResolver _someResolver;

    public SomeQuery(SomeResolver someResolver)
    {
        _someResolver = someResolver;
        Name = "some";
        Resolver = new FuncFieldResolver<Task<IReadOnlyCollection<Some>>>( Resolve);
        Type = typeof(ListGraphType<SomeGraphType>);
    }

    private async Task<IReadOnlyCollection<Some>> Resolve(ResolveFieldContext context) =>
      await base.Resolve(context, async ctx => await this.someResolver.ResolveAsync());
    }
```

So we return an error to the frontend. But how this error should look like? Should it be a string message, an error code or even maybe some translation of an error which would be shown to a customer? I decided that, because if some error occurs there is a high possibility that the call for some translations would also fail (if they are fetched from a DB/cache whatever) the best option would be to keep it as simple as it could be. So I go with returning an Error Codes. Those error codes are represented in a codebase as an enum. Which looks like this:

```csharp
public enum ErrorCode
{
    Error1,
    Error2
}
```

In this scenario, we give an information to guys which are responsible for a front-end what was happened. But because our application is localized we would want to show some decent information to a customer based on his location/culture that he chooses. Because of that, I created a new endpoint which would contain information about all available errors in our application along with their translations to the current culture. On the front-end side, we could be fetched at the startup (or when the culture would be changed) all the available translations and keep them in some static file `consts.ts` so they would be available all the time.

This endpoint looks like this:

`Source`:
```csharp
public class ErrorCodeGraphSource
{
    public ErrorCode Value { get; }
    public string Translation { get; }

    public ErrorCode(ErrorCode value, string translation)
    {
        Value = value;
        Translation = translation;
    }
}
```

`ErrorCodeEnumGraphType`:
```csharp
public class ErrorCodeEnumGraphType : EnumerationGraphType<ErrorCode>
{
    public ErrorCodeEnumGraphType()
    {
        Name = "errorCodeEnum";
    }
}
```

`GraphType`:

```csharp
public class ErrorCodeGraphType: ObjectGraphType<ErrorCodeGraphSource>
{
    public ErrorCodeGraphType()
    {
        Name = "ErrorCode";
        Field<NonNullGraphType<ErrorCodeEnumGraphType>>().Name("value").Resolve(ctx => ctx.Source.Value);
        Field<NonNullGraphType<StringGraphType>>("translation", resolve: ctx => ctx.Source.Translation);
    }
}
```

`Query`:
```csharp
internal class ErrorCodesQuery: RootQueryField
{
    private readonly IErrorCodeResolver _errorCodeResolver;

    public ErrorCodesQuery(IErrorCodeResolver errorCodeResolver)
    {
        _errorCodeResolver = errorCodeResolver;
        Name = "errorCodes";
        Resolver = new FuncFieldResolver<Task<IReadOnlyCollection<ErrorCodeGraphSource>>>(Resolve);
        Type = typeof(ListGraphType<ErrorCodeGraphType>);
    }

    private async Task<IReadOnlyCollection<ErrorCodeGraphSource>> Resolve(ResolveFieldContext context) =>
        await base.Resolve(context, async ctx => await _errorCodeResolver.ResolveAsync());
    }
```

`Resolver`:
```csharp
public interface IErrorCodeResolver
{
    Task<Result<IReadOnlyCollection<ErrorCodeGraphSource>>> ResolveAsync();
}

internal class ErrorCodeResolver : IErrorCodeResolver
{
    private readonly IErrorCodeQuery _query;

    public ErrorCodeResolver(IErrorCodeQuery query) =>
        _query = query;

    public async Task<Result<IReadOnlyCollection<ErrorCodeGraphSource>>> ResolveAsync()
    {
        var codesNames = Enum.GetNames(typeof(ErrorCode));
        var result = await _query.GetAsync(codesNames);
        return Map(result, codesNames.Length);
    }

    private static Result<IReadonlyCollection<ErrorCodeGraphSource>> Map(Result<IReadOnlyCollection<ErrorCodeTranslation>> translations) => ...
}
```

`Db Query for translations`:

```csharp
public interface IErrorCodeQuery
{
    Task<Result<IReadOnlyCollection<ErrorCodeTranslation>> GetAsync(IEnumerable<string> codes);
}

internal class ErrorCodeQuery : IErrorCodeQuery
{
    private readonly ILocalizationContext _context;

    public ErrorCodeQuery(ILocalizationContext context) =>
        _context = context;

    public async Task<Result<IReadOnlyCollection<ErrorCodeTranslation>>> GetAsync(IEnumerable<string> codes) => 
        await GetFromDb(errorCodesNames)
            .ThenAsync(Map);

    private async Task<Result<IReadOnlyCollection<ErrorCodeDb>>> GetFromDb(IEnumerable<string> codes) => ...
    private async Result<IReadOnlyCollection<ErrorCodeTranslation>> GetFromDb(Result<IReadOnlyCollection<ErrorCodeDb>> dbResult) => ...
}

```

Response to a client:

![response](https://mnie.github.com/img/19-08-2018GraphQLErrorPropagation/response.png)

As we could see thanks to such solution we could distinguish a logic behind creating and adding errors to the ExecutionErrors array from a code responsible for resolving fields. So our codebase could be still in a good shape and beyond of what we give an ability to a front-end to easily fetch/show a user-friendly message based on an gather error.

Thanks for reading!