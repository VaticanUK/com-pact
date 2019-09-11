# ComPact
An alternative Pact implementation for .NET with support for Pact Specification v3.
## Introduction
### Why another Pact implementation?
1. Most importantly, it's a fun project which allows me to learn a lot about the details of the [Pact Specification](https://github.com/pact-foundation/pact-specification).
2. My impression is that while the idea to reuse the same (Ruby) core logic for almost all Pact implementations has a lot going for it, in practice it adds quite some accidental complexity and as a result the project isn't moving forward as fast as it could. (So in my infinite hubris I've decided to take a shot at making something better.)
3. I think it's healthy for a standard/specification to have more independent implementatations of it.
 ### What's not supported
 For the foreseeable future, this implementation will not support:
* Specification versions lower than 2.0.
* Data formats other than JSON. The semantics of content-type headers and message metadata will be ignored.
* Example generators.
* "Body is present, but is null"-semantics. Due to the practicalities of .NET, no distiction will be made between a body that is non-existent and one that is null.
* Matching rules on requests.

Also note that the [DSL](#pact-content-dsl) to define a contract will not allow you to express everything that is valid within the Pact Specification. The goal is not to be complete, but to be simple and user friendly in such a way that it makes the right thing easy to do, and the wrong thing hard.

## Usage
### As an API consumer
As the consumer of an API, you'll be one to generate the Pact contract, so first you have to decide which version of the Pact Specification you want to use. To use V3, start with the following using statement in your test class:
```c#
using ComPact.Builders.V3;
```
Create a builder, and provide the base URL where the builder will create a mock provider service that you can call to verify your consumer code:
```c#
var url = "http://localhost:9393";
var builder = new PactBuilder("test-consumer", "test-provider", url);
```
Set up an interaction, which simply tells the mock provider that when it receives the specified request, it should return the specified response. Notice that the response body is described using a [DSL](#pact-content-dsl) specific for ComPact.
```c#
builder.SetupInteraction(
	new InteractionBuilder()
        .UponReceiving("a request")
        .With(Pact.Request
            .WithHeader("Accept", "application/json")
            .WithMethod(Method.GET)
            .WithPath("/testpath"))
        .WillRespondWith(Pact.Response
            .WithStatus(200)
            .WithHeader("Content-Type", "application/json")
            .WithBody(Pact.JsonContent.With(Some.Element.WithTheExactValue("test")))));
```
To test the interaction, use your *actual* client code to call the mock provider and check if it can correctly handle the response. If the actual request your code produces cannot be matched to a request that has been set up, the mock provider will return a 400 response with some explanation in the response body.

Set up any number of interactions and verify them. Finally, when you're done with your tests, create the pact contract:
```c#
await builder.BuildAsync();
```
This will verify that all interactions that have been set up have actually been triggered, and will then write a Pact json file to the project directory (by default).


## Pact Content DSL
To describe the contents of a message or the body of a response, ComPact uses a domain specific language via a fluent interface. The purpose of this is to make it easy to create a correct and useful Pact contract.