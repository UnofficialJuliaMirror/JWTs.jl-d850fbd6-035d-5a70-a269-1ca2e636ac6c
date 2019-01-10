# JSONWebTokens

[![Build Status](https://travis-ci.org/tanmaykm/JSONWebTokens.jl.svg?branch=master)](https://travis-ci.org/tanmaykm/JSONWebTokens.jl)
[![Coverage Status](https://coveralls.io/repos/tanmaykm/JSONWebTokens.jl/badge.svg?branch=master)](https://coveralls.io/r/tanmaykm/JSONWebTokens.jl?branch=master)
[![codecov.io](http://codecov.io/github/tanmaykm/JSONWebTokens.jl/coverage.svg?branch=master)](http://codecov.io/github/tanmaykm/JSONWebTokens.jl?branch=master)

JSON Web Tokens (JWT) are an open, industry standard [RFC 7519](https://tools.ietf.org/html/rfc7519) method for representing and transferring claims securely between two parties.

## Keys and Key Sets

**JWK** represents a JWK Key (either for signing or verification). JWK can be either a **JWKRSA** or **JWKSymmetric**. A RSA key can represent either the public or private key.

**JWKSet** holds a set of keys, fetched from a OpenId key URL, each key identified by a key id. The OpenId key URL is usually found in the OpenId configuration (e.g. `jwks_uri` element in <https://accounts.google.com/.well-known/openid-configuration>).

To create or verify JWT, using a JWKSet is preferred as it provides mechanism of dealing with key rotation. To refresh a JWKSet, or to load keys for the first time, call the `refresh!` method on it.

```julia
julia> using JSONWebTokens

julia> keyset = JWKSet("https://www.googleapis.com/oauth2/v3/certs")
JWKSet 0 keys (https://www.googleapis.com/oauth2/v3/certs)

julia> refresh!(keyset)

julia> keyset
JWKSet 2 keys (https://www.googleapis.com/oauth2/v3/certs)

julia> for (k,v) in keyset.keys
           println("    ", k, " => ", v.key)
       end
    7978a91347261a291bd71dcab4a464be7d279666 => MbedTLS.RSA(Ptr{MbedTLS.mbedtls_rsa_context} @0x0000000001e337e0)
    8aad66bdefc1b43d8db27e65e2e2ef301879d3e8 => MbedTLS.RSA(Ptr{MbedTLS.mbedtls_rsa_context} @0x0000000001d77390)
```

## Tokens

**JWT** represents a JSON Web Token containing the payload at the minimum. When signed, it holds the header (with key id and algorithm used) and signature too. The parts are stored in encoded form.

```julia
julia> using JSON

julia> using JSONWebTokens

julia> payload = JSON.parse("""{
           "iss": "https://auth2.juliacomputing.io/dex",
           "sub": "ChUxjfgsajfurjsjdut0483672kdhgstgy283jssZQ",
           "aud": "example-audience",
           "exp": 1536080651,
           "iat": 1535994251,
           "nonce": "1777777777777aaaaaaaaabbbbbbbbbb",
           "at_hash": "222222-G-JJJJJJJJJJJJJ",
           "email": "user@example.com",
           "email_verified": true,
           "name": "Example User"
       }""");

julia> jwt = JWT(; payload=payload)
eyJuYW1lIjoiRXhhbXBsZSBVc2VyIiwiZXhwIjoxNTM2MDgwNjUxLCJhdWQiOiJleGFtcGxlLWF1ZGllbmNlIiwic3ViIjoiQ2hVeGpmZ3NhamZ1cmpzamR1dDA0ODM2NzJrZGhnc3RneTI4M2pzc1pRIiwiaWF0IjoxNTM1OTk0MjUxLCJpc3MiOiJodHRwczovL2F1dGgyLmp1bGlhY29tcHV0aW5nLmlvL2RleCIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJhdF9oYXNoIjoiMjIyMjIyLUctSkpKSkpKSkpKSkpKSiIsIm5vbmNlIjoiMTc3Nzc3Nzc3Nzc3N2FhYWFhYWFhYWJiYmJiYmJiYmIiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20ifQ
```

A JWT can be signed using the `sign!` method, passing a key set and a key id to sign it with.

```julia
julia> issigned(jwt)
false

julia> keyset = JWKSet("file:///my/secret/location/jwkkey.json");

julia> refresh!(keyset)

julia> keyid = first(first(keyset.keys)) # using the first key in the key set
"4Fytp3LfBhriD0eZ-k3aNS042bDiCZXg6bQNJmYoaE"

julia> sign!(jwt, keyset, keyid)

julia> issigned(jwt)
true

julia> jwt # note the additional header and signature
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6IjRGeXRwM0xmQmhyaUQwZVotazNhTlMwNDJiRGlDWlhnNmJRTkptWW9hRSJ9.eyJuYW1lIjoiRXhhbXBsZSBVc2VyIiwiZXhwIjoxNTM2MDgwNjUxLCJhdWQiOiJleGFtcGxlLWF1ZGllbmNlIiwic3ViIjoiQ2hVeGpmZ3NhamZ1cmpzamR1dDA0ODM2NzJrZGhnc3RneTI4M2pzc1pRIiwiaWF0IjoxNTM1OTk0MjUxLCJpc3MiOiJodHRwczovL2F1dGgyLmp1bGlhY29tcHV0aW5nLmlvL2RleCIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJhdF9oYXNoIjoiMjIyMjIyLUctSkpKSkpKSkpKSkpKSiIsIm5vbmNlIjoiMTc3Nzc3Nzc3Nzc3N2FhYWFhYWFhYWJiYmJiYmJiYmIiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20ifQ.zfq-DT4Ft_MSU34pwFrMaealWGs0j7Ynhs9iKjf5Uf4
```

The `kid` method shows the key id used to sign a JWT. This is useful while validating a JWT.

```julia
julia> kid(jwt)
"4Fytp3LfBhriD0eZ-k3aNS042bDiCZXg6bQNJmYoaE"
```

## Validation

To validate a JWT against a key, call the `validate!` method, passing a key set and the key id to use.

The `isvalid` method can be used to check if a JWT is valid (or has been validated at all). It returns `nothing` if validation has not been attempted and a `Bool` indicating validity if it has been validated earlier.

```julia
julia> isvalid(jwt2)

julia> validate!(jwt, keyset, keyname)
true

julia> isvalid(jwt)
true
```

## TODO

Support more key algorithms. Right now it only supports RS256 and HS256.
