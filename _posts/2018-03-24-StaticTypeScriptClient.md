---
layout: post
title: Static typescript client for controller actions, is it possible?
---

Hey,

recently I interest in a topic of generating a static typescript client directly from C# controllers code. I decided that it would be a nice idea to generate such agreement in the project, in which I am currently working. For this purpose, I decided I want to try the TypeWriter library to create the contract. However, it turns out that in the case of such contract, it requires some actions which would help in gathering return types and types of accepted parameters by controllers actions.

So I started with deleting all occurrences of a parameter(s) deserialization in actions, so the signature of a function would accept parameters of concrete types, cause at this point actions looks similar to this one:

```csharp
public JsonResult Get(string data) 
{
    var d = JsonConvert.DeserializeObject<SomeType>(data);
    var r = _someService.DoSomething(d);
    return Json(r, JsonResult.AllowGet);
}
```

As we could see that accepted argument has a type of string. Which in fact hides a concrete type which is passed to an action. There were sometimes issues when we use concrete types instead of a string because passed objects are far more complicated than an object like this:

```json 
{
    "Id": 0,
    "Name": "dede"
}
```

This is because the default ASP.NET deserializer sometimes didn't work correctly. So to define concrete types I have to use ModelBinder, which gives an ability to do something with coming parameters. In our case, deserializing them by Newtonsoft.Json library. The main advantage of using the ModelBinder is that the deserialization phase took place in one place instead of every action. Implementation of such ModelBinder looks like that:

```csharp
internal class CustomModelBinder : IModelBinder
{
    private readonly Type _type;
    public CustomModelBinder(Type @type)
    {
        _type = @type;
    }

    public object BindModel(ControllerContext c, ModelBindingContext mb)
    {
        var json = controllerContext.HttpContext.Request.Params.Get(mb.ModelName);
        return json == null
            ? null
            : JsonConvert.DeserializeObject(json, _type);
    }
}
```

Of course, the implementation of ModelBinder is not enough. We have to "tell" somehow to the project, how to use this implementation. To achieve this we have to register Binder for concrete types in global.asax.cs file. Which we could do like that:

```csharp
private static readonly IReadOnlyCollection<Type> CustomBinderTypes = new[]
{
    typeof(TypeA),
    typeof(TypeB)
};

protected void Application_Start()
{
    …
    CustomBinderTypes.ForEach(t => ModelBinders.Binders.Add(t, new CustomModelBinder(t)));
    …
}
```

Thanks to above, the signature of function could evolve to the following:

```csharp
public JsonResult Get(SomeData data) {…}
```

At this moment I would know exactly what types are required by controller action parameters in a static client. But because I decided to create a static client, I also need an information about return types. When actions are defined like that they return JsonResult or ActionResult, the return type is boxed and this is not what we expect. So to return the concrete type from each action I have to implement an ActionInvoker, which has a job to do something right after the body of actions. Thanks to which the signature of an action would tell us what concrete type would be returned. But in fact somewhere below (in ActionInvoker) this value would be boxed into ActionResult or JsonResult.

Implementation of ActionInvoker looks like that:

```csharp
public class WebApiInvoker : AsyncControllerActionInvoker
{
    protected override ActionResult CreateActionResult(ControllerContext c, ActionDescriptor a, object value)
    {
        var @type = (a as ReflectedActionDescriptor)?.MethodInfo?.ReturnType;

        return typeof(ActionResult).IsAssignableFrom(@type)
            ? base.CreateActionResult(c, a, value)
            : new JsonResult(){data = value, …};
    }
}
```

Similar to the Binder, we have to tell somehow, how to use Invoker. We could achieve that in few ways. Use it in controller constructor in which we want to use it or set it in a controller factory which is responsible for creating them.

In concrete controller:

```csharp
public SomeController : Controller
{
    public SomeController()
    {
        ActionInvoker = new WebApiInvoker();
    }
}
```

In controller factory:

```csharp
internal class ServiceLocatorControllerFactory : DefaultControllerFactory
{
    protected override IController GetControllerInstance(RequestContext r, Type t)
    {
        var con = (Controller) ServiceLocator.Current.GetInstance(t);
        con.ActionInvoker = new WebApiInvoker();
        return con;
    }
}
```

Because ActionInvoker has been used for all controllers, there is a possibility, that someone still wants to return ActionResult or JsonResult for some reason. What if someone creates a function which has a return type of ActionResult od JsonResult and returns null inside of it? What is the expected behavior for this type of return types? The expected behavior is different, in case of ActionResult, the new EmptyResult should be returned in case of JSON a null as a JSON. So when we look at the implementation of WebApiInvoker right now it seems to be invalid, cause when we return null it will return a null as a JsonResult what could cause such errors:

![img](https://mnie.github.com/img/24-03-2018StaticClient/image.png)

So if we want to avoid such errors we have to check what type is expected to be returned. How could we achieve that? By reflection? Of course, but we could do it slightly easier, what could be seen here:

```csharp
protected override ActionResult CreateActionResult(ControllerContext c, ActionDescriptor a, object value)
{
    var @type = (a as ReflectedActionDescriptor)?.MethodInfo?.ReturnType;

    return typeof(ActionResult).IsAssignableFrom(@type)
        ? base.CreateActionResult(c, a, value)
        : new JsonResult(){data = value …};
}
```

We could see here that thanks to code like that, we handle all cases and the code is still clear and readable, cause in comparison to the first result, the corrected solution contains only a check what return type is expected.

At this moment I was ready to generate a contract. To achieve that goal I use a TypeWriter library and I based on a basic template.

You could install TypeWriter as a NuGet package like that:

```powershell
Install-Package TypeWriter
```
TypeWriter templates are located in .tst files. Below is a basic template to generate a contract for actions from controllers.

```typescript
${
    using Typewriter.Extensions.WebApi;
}

import { CallService } from "../core/core.module";
module App { $Classes(:Controller)[
    export class $Name {
        constructor(private http: CallService) {
        } $Methods[
        
        public $name = ($Parameters[$name: $Type][, ]) => {
            return this.http.$HttpMethod(`$Url`, $RequestData);
        }]
    }]
}
```

This code should be located in our web project.

After saving a template file. The contract should be generated and all .ts files should be attached to the template (.tst file). When we look at the generated files, we could notice a couple of things that need improvement. The first one is that this template generates a contract for all controllers. This behavior is not intentional in my case because some of the controllers are not called from a typescript code. So I want to generate contract only for files located in namespace *.Public.*.

This issue could be fixed by code like that:

```typescript
module App { $Classes(*.Public.*Controller)[
    export class $Name {
        constructor(private http: CallService) {
        } $Methods[
        
        public $name = ($Parameters[$name: $Type][, ]) => {
            return this.http.$HttpMethod(`$Url`, $RequestData);
        }]
    }]
}
```

After a while, I noticed that I would like to exclude some specific controllers from the generation. For this purpose, I wrote a function which should be located in curly braces. This piece of code is not a rocket science, its only check if the namespace of a controller starts with *.Public.* and if the file is not this one which should be excluded from generation.

```csharp
${
    using Typewriter.Extensions.WebApi;
    static string[] NotIncluedeControllers = new []{"Export", "Diagnostics", "Download"};

    bool ShouldScanController(Class c) {
        return c.FullName.StartsWith("App.Web.Controllers.Public", StringComparison.OrdinalIgnoreCase)
        && !NotIncluedeControllers.Any(x => c.Name.StartsWith(x, StringComparison.OrdinalIgnoreCase));  
    }
}
```

Invocation of this method looks like this:

```typescript
module App { $Classes($ShouldScanController)[
    export class $Name {
        constructor(private http: CallService) {
        } $Methods[
        
        public $name = ($Parameters[$name: $Type][, ]) => {
            return this.http.$methodType(`$Url`, $RequestData);
        }]
    }]
}
```

Next thing is that methods in C# code sometimes have the same names, but they differ in parameters and route, it is okay for C# code, but typescript code where a class contains methods with the same names wouldn't compile. So because of that, I decided that methods in typescript contract should have names corresponding to the last part of a route. To generate such names, we need to write a function like that:
 
```csharp
string NameFromRoute(Method m) 
{
    return m.Route()
        .Split(new string[]{"/"}, StringSplitOptions.RemoveEmptyEntries)
        .Last();
}
```
 
And use it in the template like that:
 
```typescript
$Classes($ShouldScanController)[$ImportStatements

@Injectable()
export class $Name {
    constructor(private http: CallService) {
    } $Methods[
        
    public $NameFromRoute = ($Parameters[$name: $Type][, ]): Observable<$Type> => {
        return this.http.$MethodType(`$Url`, $RequestData);
    }]
}]
```

We could see, that everything should work fine until someone defines a route like that `/get/{userId}`. We rather do not want to have a function with name `{userId}`. So to easily go around such problems, for such actions the name of typescript function will match C# method name, the fixed code looks like this:

```csharp
string NameFromRoute(Method m) 
{
    var lastRoute = m.Route()
        .Split(new string[]{"/"}, StringSplitOptions.RemoveEmptyEntries)
        .Last();
    if(lastRoute.Contains("{"))
        return m.Name;
    return lastRoute;
}
```
 
Another thing that should be fixed is getting the type of an action, when it is defined via several HttpVerbs options, the generated method looks like that:

```typescript
this.http.system.net.http.verbs.get system.net.http.verbs.post
```

It was not exactly what I expected, therefore I wrote a very simple code which, when method accepts verbs instead of concrete HttpType it should return the last specified verb (since in case of my project if method contains a couple of verbs it is always a get and then post and from a typescript code we use only a post one)
 
```csharp
string MethodType(Method m) {
    return m.HttpMethod().Split(' ').Select(x => x.Split('.').Last()).Last();
}
```

The next thing which I spot is a strange way of gathering an action type. For a method which looks like that:

```csharp
[Route(„delete”)]
public bool Delete(int id) {…}
```

The type of HttpMethod has not been set as HttpGet but as a HttpDelete (the question is why Delete function is a HttpGet ;)?). Of course, this is a bug in a code, but the behavior of generator is weird. It is a possibility that there is also some bug in TypeWriter library, although a solution to that problem was to set a proper Http type to an action in C# code (in my case HttpPost...).

The last thing which we need is to add all necessary imports at the top of each generated file. To do that I have to write a function which scans all return types, accepted parameters types, generic types and for all of them generates import statements. In our case it is a little bit simplified because a contract for classes/enums is generated via another .tst file, so they are located in one place/folder. Thanks to that construction of import statement are pretty straightforward and the whole method looks like that:
 
<script src="https://gist.github.com/MNie/edcdf5b9b8fa9281400752627cff64ae.js"></script>

And finally a full template looks like that: 

<script src="https://gist.github.com/MNie/ba0cdc9ab7423dcd29881c2da04aeb11.js"></script>

Summarizing, thanks to the above steps, controllers in our code base are much more clear and readable and don't contain unnecessary things like explicit deserialization. Also, we are able to generate a static contract for typescript code, so we could avoid hardcoding paths to actions in typescript code.

Thank you for reading :)
