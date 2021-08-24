# Evolution of C# Azure SDKs with Protocol Methods

## Overview

This document outlines the strategy for generating and evolving Azure SDKs which use *protocol methods* a new feature we are introducing.

### Protocol Methods

When a new service is onboarded, we generate an SDK which only contains *protocol methods*.  Protocol methods eschew model types (which are difficult to version and require a great deal of upfront design work to feel idiomatic in .NET) but do provide some level of convenience when compared to interacting with the service directly over HTTP.

When generating the protocol method for an operation defined in swagger, we employ the following strategy:

1. Collect all operation parameters which are annotated as required in swagger which do not appear in the body.  These are the *required parameters* and become the first set of parameters to the generated operation method. The ordering in the parameter list matches the ordering of the parameter in the swagger.
2. If an operation has any parameters which are passed in the body of the request, add a `RequestContent` parameter named `body`.
3. Collect all operation parameters which are not annotated as required in the swagger which do not appear in the body.  These are the *optional parameters* and emitted as optional parameters in the generated method.  The ordering in the parameter list matches the ordering of the parameter in the swagger
4. Add an optional `RequestOptions` parameter, named `requestOptions`, which has a default value of `default`.
5. The return value for a protocol method is always of type `Response`.

For example, let's consider a set of operations from a hypothetical Pet Store SDK:

```C#
// GET /pets/{id}
public virtual Response GetPet(string id, RequestOptions requestOptions = default);

// PUT /pets/{id}
public virtual Response CreateOrUpdatePet(string id, RequestContent body, RequestOptions requestOptions = default);

// GET /pets/petsByName?petName={petName}[&sort=<sort>&limit=<limit>&skip=<skip>]
public virtual Response GetPetsByName(string petName, string sort = "ASC", int? limit = 20, int? skip = 0, RequestOptions requestOptions = default);
```

Note that in the above example the `sort` parameter could be defined the swagger as an enumeration, however we **do not** generate a model type for it. Instead, we just use the underlying type, in this case `string`.

## Evolution

Since .NET Developers do not expect breaking changes across versions of a library, once a stable version of of an SDK has shipped, we must evolve it in a compatible manner. There are two classes of breaking changes to consider: *binary breaking changes* which prevents code which is complied against an older version of a library from running on a newer version of the library and *source breaking changes* which are changes which prevent code written against and older version of a library from compiling against a newer version of the library.  While binary breaking changes are always unacceptable, we will allow some level of source breaking changes in certain cases, a strategy also employed by the .NET Framework team.

We envision our SDKs evolving for two categories of reasons: changes in the underlying service API and the addition of one or more "convenience methods" for existing operations.

### Service Driven Evolution

As the service continues to evolve, they will add new versions of their REST API which will add new operations or augment existing operations. The Azure organization has a strong commitment to preventing breaking changes either within a single API version as well as for a specific operation which is shared across multiple API versions. Because of this strong commitment, we do not consider how any REST API breaking changes impact our generated clients.  We expect very few of these breaking changes going forward and for cases where there is a breaking change, we expect that a developer will need to analyze the impact and come up with a strategy specific to that break.

Let's consider how some classes of changes the service and how the SDK evolves with these changes.

#### New Operations

When the service adds a new operation, we generate a method for it using our default strategy. This operation will have a new name and so it will be generated as a new method on the SDK, so there is no impact to existing methods.

#### Changes to Response Body

Protocol methods do not inspect the body of a response (just the status code) and so any changes made by the service team have no impact on the SDK itself.

#### Changes to the Request Body

Protocol methods do not inspect the body of a request so any changes the service team makes to the shape of the body of a request do not have any impact on the SDK itself.

#### Adding New Optional Parameters

Since optional parameters (which do not appear in the body) are emitted as formal parameters on the operation method, the introduction of a new optional parameter from the service does impact the generated SDK. As a motivating example, let's consider how the `GetPetsByName` shown above evolves when a new optional parameter, `species` is added, which allows filtering of pets returned by the operation.

Recall that in V1 of the SDK, we have the following:

```C#
public virtual Response GetPetsByName(string petName, string sort = "ASC", int? limit = 20, int? skip = 0, RequestOptions requestOptions = default);
```

For binary compatibility, we must retain a method with this signature.  The normal .NET Framework pattern for adding a new optional parameter is to replace the existing method with a version that has no optional parameters and then add a new overload which has all the old parameters plus the new optional parameter.  For example:

```C#
/* Here for binary compatibility, calls the new overload */
public virtual Response GetPetsByName(string petName, string sort, int? limit, int? skip, RequestOptions requestOptions);

/* The new overload */
public virtual Response GetPetsByName(string petName, string sort = "ASC", int? limit = 20, int? skip = 0, string species = null, RequestOptions requestOptions = default);
```

We can use this strategy to add new optional parameters to methods.  Note that since the last parameter of a method is always of type `RequestOptions` we cannot run into an issue where the new overload creates an ambiguity between different overloads.

The only downside of this approach is the number of overloads in the method group continues to grow and it make be hard to understand which is the "core method" and which overloads which simply delegate to this core method.  One strategy we are considering is that the first time we add new optional parameters to a method, we instead add an overload like the following:

```C#
/* Here for binary compatibility, calls the new overload */
public virtual Response GetPetsByName(string petName, string sort, int? limit, int? skip, RequestOptions requestOptions);

/* The new overload */
public virtual Response GetPetsByName(string petName, RequestParameters parameters, RequestOptions requestOptions = default);
```

This design is predicated on a new `RequestParameters` type, added to Azure Core, which is a property bag mapping string names to values. Logically, this type is similar to `Dictionary<string, object>` but is optimized for holding a small number of key value pairs.

Note that the `RequestParameters` type is not specific to a specific operation, which means it will not have members that correspond to parameters for the operation and so developers will have to depend on documentation to understand the properties the service supports and their names as well as their types.

*(N.B: We have not closed on which of the two strategies we will employ, and the `RequestParameters` strategy would require some work in Azure Core, fortunately, we can start with the first strategy of just adding overloads with more and more optional parameters and then cut over to the new plan if we are comfortable with it.)*

### Developer Driven Evolution

In addition to service driven changes, the developers of an SDK can also drive evolution of the SDK by adding additional surface area to the SDK by hand. We leverage partial classes during code generation to allow a developer to add code to the client which is not auto generated.  We expect the most common form of developer driven evolution will be to add hand authored models as inputs and outputs for operations.  For example, let's imagine what happens when we want to add a `Pet` model to our client:

```C#
// Protocol Methods in V1
public virtual Response GetPet(string id, RequestOptions requestOptions = default);
public virtual Response CreateOrUpdatePet(string id, RequestContent body, RequestOptions requestOptions = default);

// Hand authored overloads
public virtual Response<Pet> GetPet(string id, CancellationToken cancellationToken = default);
public virtual Response CreateOrUpdatePet(string id, Pet pet, CancellationToken cancellationToken = default);
```

The evolution pattern here is to replace `RequestOptions` with `CancellationToken` in method signature, replace `RequestContent` with a model type that represents the body of the operation and to use `Response<T>` with a model type for operations which return values.

These additions cannot impact binary compatibility (as the old overloads remain) and by changing `RequestOptions` to `CancellationToken` we ensure that a hand authored method would never have the same signature as an existing protocol method.  The use of optional parameters does mean the following can introduce source breaking changes for methods which do not take a body. For example, consider the following code, written using a protocol method:

```C#
Response r = client.GetPet("ABC123");  // <- now ambiguous between GetPet(string, RequestOptions) and GetPet(string, CancellationToken)
string petName = JsonDocument.Parse(r.Content).Root.GetProperty("name").GetString();
```

The addition of the overload that returns `Response<T>` now creates ambiguity on which overload should be picked.  This can be resolved by adding an explicit cast:

```C#
Response r = client.GetPet("ABC123", default(RequestOptions));
string petName = JsonDocument.Parse(r.Content).Root.GetProperty("name").GetString();
```

We also plan to add `RequestOptions.Default`, allowing developers to write code like the following:

```C#
Response r = client.GetPet("ABC123", RequestOptions.Default);
string petName = JsonDocument.Parse(r.Content).Root.GetProperty("name").GetString();
```

Code written in this style would not create source ambiguities when the new overload is added.
