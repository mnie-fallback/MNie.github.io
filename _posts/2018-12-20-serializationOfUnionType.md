---
layout: post
title: Serialization of F# UnionType
---

Hi,

last time I decided to rewrite one of the services from C# to F#. One of the problems was to keep the integrity in terms of contract generated via this project. 
Previously the contract contains an enum field which I decided to represent as a union type in F#. By default Newtonsoft.Json which we are using to serialize data, serialize differently enum from C# and union type from F#. 
So going to the details of a problem. The contract looked like this: 

 
```csharp
public enum SomeEnum 
{ 
    Value1,
    Value2
}

public class Contract 
{ 
    public string Name { get; set; }
    public SomeEnum Type { get; set; }
}
```


The serialized contract, which were consumed by a front-end app looked like this: 

```javascript
{ 
    “Type”: “value1”,
    “Name”: “name”,
}
```

Rewriting an object to F# wasn’t a hard task. As I said before I decided that enum would be represented as a union type and a class as a Type. So the code in F# looked like this: 
 
```fsharp
type SomeEnum = Value1 | Value2 

type Contract = 
    { 
        Name: string 
        Type: SomeEnum 
    } 
```

The problem appears during the serialization of this type and passing it to a front-end. Because the serialized object looked like this: 
 
```javascript
{ 
    “Type”: { “Case”: “value1” } 
    “Name”: “name” 
} 
```

So as we could see this json looks a little bit different than the previous one. Because of that I decided to write my own converter for union types. The first thing was a serialization. 
What I want to achieve in the serialized object is information only about the value of union type as a string. But this value has to start with a lower case. Keeping in mind what I plan about that here is a piece of code to do that: 
 
```fsharp
type EnumAsStringConverter () = 
    inherit JsonConverter() 
    
    override this.CanConvert objectType = FSharpType.IsUnion(objectType) 
    override this.WriteJson (writer: JsonWriter, value: obj, serializer: JsonSerializer): unit = 
        if value = null then writer.WriteValue value 
        else 
            let str = value.ToString() 
            let firstLetter = Char.ToLowerInvariant(str.[0]) 
            let theRest = str.Substring(1) 
            writer.WriteValue(sprintf "%c%s" firstLetter theRest) 
```

This code has a task to get the actual union value and then invoke a ‘toString’ on it. But keeping in mind that it has to start with a lower case. So I run my tests which checks a compatibility with the old json and everything seems to work fine. 
But here appeared a problem. What with deserialization to a union type? Right now there is no possibility to deserialize string to union. To do that I override a read method which looks right now like this: 

```fsharp
...
override this.CanRead = true 

override this.ReadJson (reader: JsonReader, objectType: Type, existingValue: obj, serializer: JsonSerializer) : obj = 
    if reader.TokenType <> JsonToken.String then 
        (sprintf "Cannot deserialize %A type to Union" reader.TokenType) |> ArgumentException |> raise 
    else 
        if FSharpType.IsUnion (objectType, BindingFlags.Public) then 
            let matchedCase = FSharpType.GetUnionCases(objectType, BindingFlags.Public) |> Array.tryFind (fun x -> x.Name.ToLower() = (reader.Value :?> string).ToLower()) 
            match matchedCase with 
            | Some s -> FSharpValue.MakeUnion(s, [||]) 
            | None -> (sprintf "Cannot find maching case %A in union case: %A" reader.Value objectType) |> ArgumentException |> raise 
        else (sprintf "objectType is not an union, instead it is a: %A" objectType) |> ArgumentException |> raise 
```

To create an ACL at first I checked if the value which I want to deserialize is a string and if the type to which I want to deserialize is a union type. Then I looked in all of the fields of a union and check if it has a value like this ignoring the case. 
If I found one I create a union with the following value. I run tests one more time and everything is green.  

To sum up moving parts of C# code to F# with keeping the backward compatibility in a contract could be problematic. And sometimes we have to do some extra work to achieve this. 
But still, I think that because of easy writing programs in F# in comparison to C# sometimes it is worth to take a while to look and do this. 

Thanks for reading!
