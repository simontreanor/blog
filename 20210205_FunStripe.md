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

### Record types

### Discriminated unions

### `Option` types

### Optional parameters

### Computation expression

### F# Interactive

## Stripe API Ideosyncracies

Some of the difficulties in developing for the Stripe API include:

- Request bodies need to be supplied as form values rather than JSON
- Requests use a mixture of path, query-string and form parameters
- Stripe is inconsistent in that some enumerations are represented as strings and others as objects
- Some fields in responses are polymorphic, e.g. a customer can be returned as either a string representing the customer ID or as a full customer object
- Some aspects of the workflow require the use of client script (e.g. collecting card payment details) to protect customer confidentiality (this is not handled by the
FunStripe library, though I will shortly publish a [Bolero](https://fsbolero.io/)-based app to show how easily it can be done
- Inconsistent naming conventions causing issue for serialisation
- Paged lists

Again, let's look at each of these in a bit more detail.

### Form-value requests

### Parameters

### Enumerations

### Client-side scripting

### Polymorphism

### Serialisation

### List values
