Simon Treanor, 2021-02-05 Friday

# FunStripe

FunStripe is an F# 5.0 library to connect to the [Stripe API](https://stripe.com/docs/api), including code generators to update the model and requests. The respository is
[here](https://github.com/simontreanor/FunStripe).

Though there is already an [official Stripe .NET library](https://github.com/stripe/stripe-dotnet), I wanted to create one in F# with a cleaner, more functional approach,
taking advantage of strong typing with records and discriminated unions.

## Background

Stripe publishes an [OpenAPI specification](https://raw.githubusercontent.com/stripe/openapi/master/openapi/spec3.sdk.json), which it is possible to consume using a type
provider, but the issue with that is that it is such a large file that it takes quite some time to parse it on the fly. Hence the idea of code generators to do the parsing
up front and create static F# files for the model and requests. There are a couple of nice advantages to this approach:

- Performance improvements
- The code generators can be tweaked by developers to deal with the Stripe API's ideosyncracies in whatever way they see fit
- When the API version is updated, new model and request files can be generated and the diff inspected to see what impact the version change may have on existing code

## F#-specific Features

- Modules and submodules to group related functions and provide name disambiguation
- Record types to represent objects
- Discriminated unions to represent string enumerations
- `Option` types to specifically represent nullable values
- `?` to represent optional parameters in static constructor methods for records
- Custom computation expression to help with error handling
- Use of F# Interactive to generate code files

Let's look at each of these in a bit more detail.

### Structure

- Json\*: an extract of `FSharp.Json` modified for Stripe
- `AsyncResult.fs`: custom computation expression (see below)
- `Config.fs`: some global attributes plus user settings, including a function to retrieve the Stripe API key from [UserSecrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-5.0&tabs=windows)
- `FormUtil.fs`: functions for formatting request bodies as form values
- `Program.fs`: test functions for debugging
- `RestApi.fs`: simple REST API client and error parser
- `StripeError.fs`: definition of Stripe error object
- `ModelBuilder.fs` and `RequestBuilder.fs`:

These two files contain code to generate the model and request code files using F# Interactive (see below).

The two files generated are `StripeModel.fs` and `StripeRequest.fs`. The model contains all of the Stripe objects that are returned by the requests. They are separate files due to the complexity of the API and the  different nature of the information in each of them.

The `StripeModel` module is not separated into submodules due to the number of cross references between the record types. In F# types must normally be declared before they are used, but the nature of the Stripe API model is so complex that this was not practical. Instead, a recursive type notation is used, with the keyword `type` declaring the first type and the keyword `and` declaring all subsequent types:

```fsharp
module StripeModel =
    type Account = {
        //…
    }
    and AccountBusinessType =
    |   //…
```

The `StripeRequest` module is separated into submodules, for two main reasons: firstly, there are a lot of name clashes, as there are lots of `Create` and `Retrieve` functions for example; and secondly, the types and functions within each module do not have any other dependencies elsewhere in the `StripeRequest` module, only relying on the `StripeModel` module to be opened. Each submodule is therefore self-contained and always follows the same structure: and Options type followed by a function taking the options as a parameter:

```fsharp
module StripeRequest =
    //…
    module Account =
        type RetrieveOptions = {
            //…
        }
            //…
        let Retrieve settings (options: RetrieveOptions) =
            //…
```

### Record types

In the Stripe API model and request modules many types have a large number of properties and so instantiating records using all their properties is not practical. So each record is declared in the normal way but with a static function appended to create an instance of that record type:

```fsharp
    and AccountDashboardSettings = {
        DisplayName: string option
        Timezone: string option
    }
    with
        static member New (displayName: string option, timezone: string option) =
            {
                AccountDashboardSettings.DisplayName = displayName //required
                AccountDashboardSettings.Timezone = timezone //required
            }
```

### Discriminated unions

String enumerations are represented wherever possible using discriminated unions. These have the advantage of being strongly typed and therefore help prevent coding errors:

```fsharp
    and AccountBusinessType =
        | Company
        | GovernmentEntity
        | Individual
        | NonProfit
```

### Option types and optional parameters

These two concepts are closely related. Parameter type declarations are marked with the `option` keyword, while optional functional parameters are marked by a `?` prepended to the parameter name. Sometimes parameters are both optional and `Option` types, in which case the option needs flattening when assigning the value:

```fsharp
    and AccountTosAcceptance = {
        Date: DateTime option
        Ip: string option
        ServiceAgreement: string option
        UserAgent: string option
    }
    with
        static member New (?date: DateTime option, ?ip: string option, ?serviceAgreement: string, ?userAgent: string option) =
            {
                AccountTosAcceptance.Date = date |> Option.flatten
                AccountTosAcceptance.Ip = ip |> Option.flatten
                AccountTosAcceptance.ServiceAgreement = serviceAgreement
                AccountTosAcceptance.UserAgent = userAgent |> Option.flatten
            }
```

### Computation expression

The `AsyncResultCE` module defines a custom computation expression, `asyncResult`. This is a way to combine async calls with `Result` return values:

```fsharp
    type AsyncResult<'ok,'error> = Async<Result<'ok,'error>>
```

This is used e.g. as follows:

```fsharp
let result =
    asyncResult {
        let expected = defaultPaymentMethod // PaymentMethod
        let! actual = getNewPaymentMethod() // unit -> AsyncResult<PaymentMethod,ErrorResponse>
        return expected, actual
    }
    |> Async.RunSynchronously
match result with
| Ok (exp, act) ->
    //…
| Error e ->
    //…
```

In practical terms what this does is enable you to deal with the actual values within the computation expression (`ayncResult { … }`) and not worry about writing conditions to check for errors at each step. The code proceeds happily along provided that the result is `Ok`, but if at any point something goes wrong it will stop processing and return a result of `Error`.

### F# Interactive

`ModelBuilder.fs` and `RequestBuilder.fs` are both essentially normal F# modules wrapped in a bit of code to enable the code to be sent to F# Interactive:

```fsharp
#if INTERACTIVE
    #r "nuget: FSharp.Data";;
#else
namespace FunStripe
#endif
//…
#if INTERACTIVE
    ;;
    open ModelBuilder;;
    let s = parseModel None;;
    System.IO.File.WriteAllText(__SOURCE_DIRECTORY__ + "/StripeModel.fs", s);;
#endif
```

In VS Code, selecting the entire code in the file and pressing `Alt` + `Enter` sends the code to F# Interactive. The `;;` causes the code to run, after which a function parses the API specification and outputs the `.fs` code file.

## Stripe API Ideosyncracies

Some of the difficulties in developing for the Stripe API include:

- Request bodies need to be supplied as form values rather than JSON
- Requests use a mixture of path, query-string and form parameters
- Stripe is inconsistent in that some enumerations are represented as strings and others as objects
- Some fields in responses are polymorphic, e.g. a customer can be returned as either a string representing the customer ID or as a full customer object
- Some aspects of the workflow require the use of client script (e.g. collecting card payment details) to protect customer confidentiality (this is not handled by the
FunStripe library, though I will shortly publish a [Bolero](https://fsbolero.io/)-based app to show how easily it can be done
- Inconsistent naming conventions causing issue for serialisation
- DateTime values represented as Unix timestamps
- Paged lists
- Mixture of non-required and nullable fields

Again, let's look at each of these in a bit more detail.

### Form-value requests

Rather strangely, despite responding in JSON to requests, body parameters are not formatted using JSON but rather are x-www-form-urlencoded, .e.g.:

```fsharp
seq {
    "type", "card"
    "card[number]", "4242424242424242"
    "card[exp_month]", "10"
    "card[exp_year]", "2021"
    "card[cvc]", "314"
}
```

In order to handle this the `FormUtil` module uses some functions derived from the C# version in Stripe.NET, with modifications to anable it to handle discriminated unions, `Option` types, `Choice` types and records.

### Parameters

Stripe request parameters can be either in the path, the query string or the body. To simplify requests these three types of parameter are all concatenated into a single option record type. Parameters are typically a mix of path and query for get requests and path and form for post requests, and are distiguished by attributes, e.g.:

```fsharp
    type RetrieveOptions = {
        [<Config.Path>]Account: string
        [<Config.Query>]Expand: string list option
    }
    //…
    type Update'BusinessProfileSupportAddress = {
        [<Config.Form>]City: string option
        [<Config.Form>]Country: string option
        //…
    }
```

### Enumerations

In many cases the Stripe API provides the possible values for an enumeration in the specification. In other cases however, the possible values are only provided in the description for the parameter. If explicit values are not provided, the model builder parses them from the text of the description. They are then represented as discriminated unions, with the huge benefit of being strongly typed.

### Client-side scripting

### Polymorphism

### Serialisation

### List values
