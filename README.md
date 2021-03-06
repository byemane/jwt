﻿[![Build status](https://abatishchev.visualstudio.com/OpenSource/_apis/build/status/Jwt.Net-CI)](https://abatishchev.visualstudio.com/OpenSource/_build/latest?definitionId=7)
[![Release status](https://abatishchev.vsrm.visualstudio.com/_apis/public/Release/badge/b7fc2610-91d5-4968-814c-97a9d76b03c4/2/2)](https://abatishchev.visualstudio.com/OpenSource/_release?_a=releases&view=mine&definitionId=2)

# Jwt.Net, a JWT (JSON Web Token) implementation for .NET

This library supports generating and decoding [JSON Web Tokens](https://tools.ietf.org/html/rfc7519).

## Avaliable packages

1. [Jwt.Net](#JwtNet)
2. [Jwt.Net for ASP.NET Core](#JwtNet-ASPNET-Core)
3. [Jwt.Net for Owin](#JwtNet-OWIN)

## Supported .NET versions:

- .NET Framework 4.6.0
- .NET Framework 4.7.2
- .NET Standard 1.3
- .NET Standard 2.0

## License

The following projects and their resulting packages are licensed under Public Domain, see the [LICENSE#Public-Domain](LICENSE.md#MIT) file.

- JWT 

The following projects and their resulting packages are licensed under the MIT License, see the [LICENSE#MIT](LICENSE.md#MIT) file.

- JWT.Extensions.AspNetCore

In addition the maintainer ([@abatishchev](https://github.com/abatishchev)) of this repository also shares the values of the [Hippocratic License](https://firstdonoharm.dev/version/1/1/license.txt).

### Jwt.NET

#### NuGet

[![NuGet](https://img.shields.io/nuget/v/JWT.svg)](https://www.nuget.org/packages/JWT)
[![NuGet Pre](https://img.shields.io/nuget/vpre/JWT.svg)](https://www.nuget.org/packages/JWT)

#### Creating (encoding) token

```c#
var payload = new Dictionary<string, object>
{
    { "claim1", 0 },
    { "claim2", "claim2-value" }
};
const string secret = "GQDstcKsx0NHjPOuXOYg5MbeJ1XT0uFiwDVvVBrk";

IJwtAlgorithm algorithm = new HMACSHA256Algorithm();
IJsonSerializer serializer = new JsonNetSerializer();
IBase64UrlEncoder urlEncoder = new JwtBase64UrlEncoder();
IJwtEncoder encoder = new JwtEncoder(algorithm, serializer, urlEncoder);

var token = encoder.Encode(payload, secret);
Console.WriteLine(token);
```

#### Or using the fluent builder API

```c#
  var token = new JwtBuilder()
      .WithAlgorithm(new HMACSHA256Algorithm())
      .WithSecret(secret)
      .AddClaim("exp", DateTimeOffset.UtcNow.AddHours(1).ToUnixTimeSeconds())
      .AddClaim("claim2", "claim2-value")
      .Build();

Console.WriteLine(token);
```

The output would be:

>eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJjbGFpbTEiOjAsImNsYWltMiI6ImNsYWltMi12YWx1ZSJ9.8pwBI_HtXqI3UgQHQ_rDRnSQRxFL1SR8fbQoS-5kM5s

#### Parsing (decoding) and verifying token

```c#
const string token = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJjbGFpbTEiOjAsImNsYWltMiI6ImNsYWltMi12YWx1ZSJ9.8pwBI_HtXqI3UgQHQ_rDRnSQRxFL1SR8fbQoS-5kM5s";
const string secret = "GQDstcKsx0NHjPOuXOYg5MbeJ1XT0uFiwDVvVBrk";

try
{
    IJsonSerializer serializer = new JsonNetSerializer();
    IDateTimeProvider provider = new UtcDateTimeProvider();
    IJwtValidator validator = new JwtValidator(serializer, provider);
    IBase64UrlEncoder urlEncoder = new JwtBase64UrlEncoder();
    IJwtDecoder decoder = new JwtDecoder(serializer, validator, urlEncoder);
    
    var json = decoder.Decode(token, secret, verify: true);
    Console.WriteLine(json);
}
catch (TokenExpiredException)
{
    Console.WriteLine("Token has expired");
}
catch (SignatureVerificationException)
{
    Console.WriteLine("Token has invalid signature");
}
```

#### Or using the fluent builder API

```c#
try
{
    var json = new JwtBuilder()
        .WithSecret(secret)
        .MustVerifySignature()
        .Decode(token);                    
    Console.WriteLine(json);
}
catch (TokenExpiredException)
{
    Console.WriteLine("Token has expired");
}
catch (SignatureVerificationException)
{
    Console.WriteLine("Token has invalid signature");
}
```

The output would be:

>{ "claim1": 0, "claim2": "claim2-value" }

You can also deserialize the JSON payload directly to a .NET type:

```c#
var payload = decoder.DecodeToObject<IDictionary<string, object>>(token, secret);
Console.WriteLine(payload["claim2"]);
 ```

#### Or using the fluent builder API

```c#
var payload = new JwtBuilder()
        .WithSecret(secret)
        .MustVerifySignature()
        .Decode<IDictionary<string, object>>(token);     
Console.WriteLine(payload["claim2"]);
```

The output would be:
    
>claim2-value

#### Set and validate token expiration

As described in the [JWT RFC](https://tools.ietf.org/html/rfc7519#section-4.1.4), the `exp` "claim identifies the expiration time on or after which the JWT MUST NOT be accepted for processing." If an `exp` claim is present and is prior to the current time the token will fail verification. The exp (expiry) value must be specified as the number of seconds since 1/1/1970 UTC.

```csharp
IDateTimeProvider provider = new UtcDateTimeProvider();
var now = provider.GetNow();

var secondsSinceEpoch = UnixEpoch.GetSecondsSince(now);

var payload = new Dictionary<string, object>
{
    { "exp", secondsSinceEpoch }
};
const string secret = "GQDstcKsx0NHjPOuXOYg5MbeJ1XT0uFiwDVvVBrk";
var token = encoder.Encode(payload, secret);

var json = decoder.Decode(token, secret); // throws TokenExpiredException
```

#### Custom JSON serializer

By default JSON serialization is performed by JsonNetSerializer implemented using [Json.Net](https://www.json.net). To use a different one, implement the `IJsonSerializer` interface:

```c#
public class CustomJsonSerializer : IJsonSerializer
{
    public string Serialize(object obj)
    {
        // Implement using favorite JSON serializer
    }

    public T Deserialize<T>(string json)
    {
        // Implement using favorite JSON serializer
    }
}
```

And then pass this serializer to JwtEncoder constructor:

```c#
IJwtAlgorithm algorithm = new HMACSHA256Algorithm();
IJsonSerializer serializer = new CustomJsonSerializer();
IBase64UrlEncoder urlEncoder = new JwtBase64UrlEncoder();
IJwtEncoder encoder = new JwtEncoder(algorithm, serializer, urlEncoder);
```

#### Custom JSON serialization settings with the default JsonNetSerializer

As mentioned above, the default JSON serialization is done by `JsonNetSerializer`. You can define your own custom serialization settings as follows:

```c#
JsonSerializer customJsonSerializer = new JsonSerializer
{
    // All keys start with lowercase characters instead of the exact casing of the model/property, e.g. fullName
    ContractResolver = new CamelCasePropertyNamesContractResolver(), 
    
    // Nice and easy to read, but you can also use Formatting.None to reduce the payload size
    Formatting = Formatting.Indented,
    
    // The most appropriate datetime format.
    DateFormatHandling = DateFormatHandling.IsoDateFormat,
    
    // Don't add keys/values when the value is null.
    NullValueHandling = NullValueHandling.Ignore,
    
    // Use the enum string value, not the implicit int value, e.g. "red" for enum Color { Red }
    Converters.Add(new StringEnumConverter())
};
IJsonSerializer serializer = new JsonNetSerializer(customJsonSerializer);
```

### Jwt.Net ASP.NET Core

#### NuGet

[![NuGet](https://img.shields.io/nuget/v/JWT.Extensions.AspNetCore.svg)](https://www.nuget.org/packages/JWT.Extensions.AspNetCore)
[![NuGet Pre](https://img.shields.io/nuget/vpre/JWT.Extensions.AspNetCore.svg)](https://www.nuget.org/packages/JWT.Extensions.AspNetCore)

#### Register authentication handler to validate JWT

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(options =>
                 {
                     options.DefaultAuthenticateScheme = JwtAuthenticationDefaults.AuthenticationScheme;
                     options.DefaultChallengeScheme = JwtAuthenticationDefaults.AuthenticationScheme;
                 })
            .AddJwt(options =>
                 {
                     // secrets
                     options.Keys = new[] { "GQDstcKsx0NHjPOuXOYg5MbeJ1XT0uFiwDVvVBrk" };
                     
                     // force JwtDecoder to throw exception if JWT signature is invalid
                     options.VerifySignature = true;
                 });
}

public void Configure(IApplicationBuilder app)
{
    app.UseAuthentication();
}
```

#### Custom factories to produce Identity or AuthneticationTicket

```c#
options.IdentityFactory = dic => new ClaimsIdentity(
    dic.Select(p => new Claim(p.Key, p.Value)));

options.TicketFactory = (identity, scheme) => new AuthenticationTicket(
    new ClaimsPrincipal(identity),
    new AuthenticationProperties(),
    scheme.Name);
```

#### Register middleware to validate JWT

```c#
app.UseJwtMiddleware();
```

**Note:** work in progress as the scenario/usage is not designed yet. The registered will do nothing but throw an exception.


### Jwt.Net OWIN

#### NuGet

[![NuGet](https://img.shields.io/nuget/v/JWT.Extensions.Owin.svg)](https://www.nuget.org/packages/JWT.Extensions.Owin)
[![NuGet Pre](https://img.shields.io/nuget/vpre/JWT.Extensions.Owin.svg)](https://www.nuget.org/packages/JWT.Extensions.Owin)

#### Register middleware to validate JWT

```c#
app.UseJwtMiddleware();
```

**Note:** work in progress as the scenario/usage is not designed yet. The registered will do nothing but throw an exception.
