[TOC]

## JSON Web Token (JWT) Code Examples ##

### Producing and consuming a signed JWT 
And example showing simple generation and consumption of a JWT 
```
#!java
    //
    // JSON Web Token is a compact URL-safe means of representing claims/attributes to be transferred between two parties.
    // This example demonstrates producing and consuming a signed JWT
    //

    // Generate an RSA key pair, which will be used for signing and verification of the JWT, wrapped in a JWK
    RsaJsonWebKey rsaJsonWebKey = RsaJwkGenerator.generateJwk(2048);

    // Give the JWK a Key ID (kid), which is just the polite thing to do
    rsaJsonWebKey.setKeyId("k1");

    // Create the Claims, which will be the content of the JWT
    JwtClaims claims = new JwtClaims();
    claims.setIssuer("Issuer");  // who creates the token and signs it
    claims.setAudience("Audience"); // to whom the token is intended to be sent
    claims.setExpirationTimeMinutesInTheFuture(10); // time when the token will expire (10 minutes from now)
    claims.setGeneratedJwtId(); // a unique identifier for the token
    claims.setIssuedAtToNow();  // when the token was issued/created (now)
    claims.setNotBeforeMinutesInThePast(2); // time before which the token is not yet valid (2 minutes ago)
    claims.setSubject("subject"); // the subject/principal is whom the token is about
    claims.setClaim("email","mail@example.com"); // additional claims/attributes about the subject can be added
    List<String> groups = Arrays.asList("group-one", "other-group", "group-three");
    claims.setStringListClaim("groups", groups); // multi-valued claims work too and will end up as a JSON array

    // A JWT is a JWS and/or a JWE with JSON claims as the payload.
    // In this example it is a JWS so we create a JsonWebSignature object.
    JsonWebSignature jws = new JsonWebSignature();

    // The payload of the JWS is JSON content of the JWT Claims
    jws.setPayload(claims.toJson());

    // The JWT is signed using the private key
    jws.setKey(rsaJsonWebKey.getPrivateKey());

    // Set the Key ID (kid) header because it's just the polite thing to do.
    // We only have one key in this example but a using a Key ID helps
    // facilitate a smooth key rollover process
    jws.setKeyIdHeaderValue(rsaJsonWebKey.getKeyId());

    // Set the signature algorithm on the JWT/JWS that will integrity protect the claims
    jws.setAlgorithmHeaderValue(AlgorithmIdentifiers.RSA_USING_SHA256);

    // Sign the JWS and produce the compact serialization or the complete JWT/JWS
    // representation, which is a string consisting of three dot ('.') separated
    // base64url-encoded parts in the form Header.Payload.Signature
    // If you wanted to encrypt it, you can simply set this jwt as the payload
    // of a JsonWebEncryption object and set the cty (Content Type) header to "jwt".
    String jwt = jws.getCompactSerialization();


    // Now you can do something with the JWT. Like send it to some other party
    // over the clouds and through the interwebs.
    System.out.println("JWT: " + jwt);


    // Use JwtConsumerBuilder to construct an appropriate JwtConsumer, which will
    // be used to validate and process the JWT.
    // The specific validation requirements for a JWT are context dependent, however,
    // it typically advisable to require a expiration time, a trusted issuer, and
    // and audience that identifies your system as the intended recipient.
    // If the JWT is encrypted too, you need only provide a decryption key or
    // decryption key resolver to the builder.
    JwtConsumer jwtConsumer = new JwtConsumerBuilder()
            .setRequireExpirationTime() // the JWT must have an expiration time
            .setAllowedClockSkewInSeconds(30) // allow some leeway in validating time based claims to account for clock skew
            .setRequireSubject() // the JWT must have a subject claim
            .setExpectedIssuer("Issuer") // whom the JWT needs to have been issued by
            .setExpectedAudience("Audience") // to whom the JWT is intended for
            .setVerificationKey(rsaJsonWebKey.getKey()) // verify the signature with the public key
            .build(); // create the JwtConsumer instance

    try
    {
        //  Validate the JWT and process it to the Claims
        JwtClaims jwtClaims = jwtConsumer.processToClaims(jwt);
        System.out.println("JWT validation succeeded! " + jwtClaims);
    }
    catch (InvalidJwtException e)
    {
        // InvalidJwtException will be thrown, if the JWT failed processing or validation in anyway.
        // Hopefully with meaningful explanations(s) about what went wrong.
        System.out.println("Invalid JWT! " + e);
    }

```


### Using an HTTPS JWKS endpoint
If the JWT Issuer has their public keys at a HTTPS JWKS endpoint
```
#!java
    // In the example above we generated a key pair and used it directly for signing and verification.
    // Key exchange in the real word, however, is rarely so simple.
    // A common pattern that's emerging is for an issuer to publish its public keys
    // as a JSON Web Key Set at an HTTPS endpoint. And for the consumer of the JWT to periodically,
    // based on cache directives or known/unknown Key IDs, retrieve the keys from the host authenticated
    // and secured endpoint.

    // The HttpsJwks retrieves and caches keys from a the given HTTPS JWKS endpoint.
    // Because it retains the JWKs after fetching them, it can and should be reused
    // to improve efficiency by reducing the number of outbound calls the the endpoint.
    HttpsJwks httpsJkws = new HttpsJwks("https://example.com/jwks");

    // The HttpsJwksVerificationKeyResolver uses JWKs obtained from the HttpsJwks and will select the
    // most appropriate one to use for verification based on the Key ID and other factors provided
    // in the header of the JWS/JWT.
    HttpsJwksVerificationKeyResolver httpsJwksKeyResolver = new HttpsJwksVerificationKeyResolver(httpsJkws);


    // Use JwtConsumerBuilder to construct an appropriate JwtConsumer, which will
    // be used to validate and process the JWT. But, in this case, provide it with
    // the HttpsJwksVerificationKeyResolver instance rather than setting the
    // verification key explicitly.
    jwtConsumer = new JwtConsumerBuilder()
            // ... other set up of the JwtConsumerBuilder ...
            .setVerificationKeyResolver(httpsJwksKeyResolver)
            // ...
            .build();



```


### Using JWKs
Or you got some JWKs out-of-band from the JWT Issuer. 
```
#!java

    // There's also a key resolver that selects from among a given list of JWKs using the Key ID
    // and other factors provided in the header of the JWS/JWT.
    JsonWebKeySet jsonWebKeySet = new JsonWebKeySet(rsaJsonWebKey);
    JwksVerificationKeyResolver jwksResolver = new JwksVerificationKeyResolver(jsonWebKeySet.getJsonWebKeys());
    jwtConsumer = new JwtConsumerBuilder()
            // ... other set up of the JwtConsumerBuilder ...
            .setVerificationKeyResolver(jwksResolver)
            // ...
            .build();



```


### X.509 
X.509 Certificates? No problem, there's a Resolver for that too.
```
#!java
    // Sometimes X509 certificate(s) are provided out-of-band somehow by the signer/issuer
    // and the X509VerificationKeyResolver is helpful for that situation. It will use
    // the X.509 Certificate Thumbprint Headers (x5t or x5t#S256) from the JWS/JWT to
    // select from among the provided certificates to get the public key for verification.
    X509Util x509Util = new X509Util();
    X509Certificate certificate = x509Util.fromBase64Der(
            "MIIDQjCCAiqgAwIBAgIGATz/FuLiMA0GCSqGSIb3DQEBBQUAMGIxCzAJB" +
            "gNVBAYTAlVTMQswCQYDVQQIEwJDTzEPMA0GA1UEBxMGRGVudmVyMRwwGgYD" +
            "VQQKExNQaW5nIElkZW50aXR5IENvcnAuMRcwFQYDVQQDEw5CcmlhbiBDYW1" +
            "wYmVsbDAeFw0xMzAyMjEyMzI5MTVaFw0xODA4MTQyMjI5MTVaMGIxCzAJBg" +
            "NVBAYTAlVTMQswCQYDVQQIEwJDTzEPMA0GA1UEBxMGRGVudmVyMRwwGgYDV" +
            "QQKExNQaW5nIElkZW50aXR5IENvcnAuMRcwFQYDVQQDEw5CcmlhbiBDYW1w" +
            "YmVsbDCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAL64zn8/QnH" +
            "YMeZ0LncoXaEde1fiLm1jHjmQsF/449IYALM9if6amFtPDy2yvz3YlRij66" +
            "s5gyLCyO7ANuVRJx1NbgizcAblIgjtdf/u3WG7K+IiZhtELto/A7Fck9Ws6" +
            "SQvzRvOE8uSirYbgmj6He4iO8NCyvaK0jIQRMMGQwsU1quGmFgHIXPLfnpn" +
            "fajr1rVTAwtgV5LEZ4Iel+W1GC8ugMhyr4/p1MtcIM42EA8BzE6ZQqC7VPq" +
            "PvEjZ2dbZkaBhPbiZAS3YeYBRDWm1p1OZtWamT3cEvqqPpnjL1XyW+oyVVk" +
            "aZdklLQp2Btgt9qr21m42f4wTw+Xrp6rCKNb0CAwEAATANBgkqhkiG9w0BA" +
            "QUFAAOCAQEAh8zGlfSlcI0o3rYDPBB07aXNswb4ECNIKG0CETTUxmXl9KUL" +
            "+9gGlqCz5iWLOgWsnrcKcY0vXPG9J1r9AqBNTqNgHq2G03X09266X5CpOe1" +
            "zFo+Owb1zxtp3PehFdfQJ610CDLEaS9V9Rqp17hCyybEpOGVwe8fnk+fbEL" +
            "2Bo3UPGrpsHzUoaGpDftmWssZkhpBJKVMJyf/RuP2SmmaIzmnw9JiSlYhzo" +
            "4tpzd5rFXhjRbg4zW9C+2qok+2+qDM1iJ684gPHMIY8aLWrdgQTxkumGmTq" +
            "gawR+N5MDtdPTEQ0XfIBc2cJEUyMTY5MPvACWpkA6SdS4xSvdXK3IVfOWA==");

    X509Certificate otherCertificate = x509Util.fromBase64Der(
            "MIICUDCCAbkCBETczdcwDQYJKoZIhvcNAQEFBQAwbzELMAkGA1UEBhMCVVMxCzAJ" +
            "BgNVBAgTAkNPMQ8wDQYDVQQHEwZEZW52ZXIxFTATBgNVBAoTDFBpbmdJZGVudGl0" +
            "eTEXMBUGA1UECxMOQnJpYW4gQ2FtcGJlbGwxEjAQBgNVBAMTCWxvY2FsaG9zdDAe" +
            "Fw0wNjA4MTExODM1MDNaFw0zMzEyMjcxODM1MDNaMG8xCzAJBgNVBAYTAlVTMQsw" +
            "CQYDVQQIEwJDTzEPMA0GA1UEBxMGRGVudmVyMRUwEwYDVQQKEwxQaW5nSWRlbnRp" +
            "dHkxFzAVBgNVBAsTDkJyaWFuIENhbXBiZWxsMRIwEAYDVQQDEwlsb2NhbGhvc3Qw" +
            "gZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBAJLrpeiY/Ai2gGFxNY8Tm/QSO8qg" +
            "POGKDMAT08QMyHRlxW8fpezfBTAtKcEsztPzwYTLWmf6opfJT+5N6cJKacxWchn/" +
            "dRrzV2BoNuz1uo7wlpRqwcaOoi6yHuopNuNO1ms1vmlv3POq5qzMe6c1LRGADyZh" +
            "i0KejDX6+jVaDiUTAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAMojbPEYJiIWgQzZc" +
            "QJCQeodtKSJl5+lA8MWBBFFyZmvZ6jUYglIQdLlc8Pu6JF2j/hZEeTI87z/DOT6U" +
            "uqZA83gZcy6re4wMnZvY2kWX9CsVWDCaZhnyhjBNYfhcOf0ZychoKShaEpTQ5UAG" +
            "wvYYcbqIWC04GAZYVsZxlPl9hoA=");

    X509VerificationKeyResolver x509VerificationKeyResolver = new X509VerificationKeyResolver(certificate, otherCertificate);

    // Optionally the X509VerificationKeyResolver can attempt to verify the signature
    // with the key from each of the provided certificates, if no X.509 Certificate
    // Thumbprint Header is present in the JWT/JWS.
    x509VerificationKeyResolver.setTryAllOnNoThumbHeader(true);

    jwtConsumer = new JwtConsumerBuilder()
            // ... other set up of the JwtConsumerBuilder ...
            .setVerificationKeyResolver(x509VerificationKeyResolver)
            // ...
            .build();


    // Note that on the producing side, the X.509 Certificate Thumbprint Header
    // can be set like this on the JWS (which is the JWT)
    jws.setX509CertSha1ThumbprintHeaderValue(certificate);



```

### Two-pass JWT consumption
Sometimes you'll need to crack open the JWT in order to know who issued it and how to validate it, which can be done efficiently and relatively easily using a two-pass consumption approach. 
```
#!java

    // In some cases you won't have enough information to set up your JWT consumer without cracking open
    // the JWT first. For example, in some contexts you might not know who issued the token without looking
    // at the "iss" claim inside the JWT.
    // This can be done efficiently and relatively easily using two JwtConsumers in a "two-pass" validation
    // of sorts - the first JwtConsumer parses the JWT and the second one does the actual validation.

    // Build a JwtConsumer that doesn't check signatures or do any validation.
    JwtConsumer firstPassJwtConsumer = new JwtConsumerBuilder()
            .setSkipAllValidators()
            .setDisableRequireSignature()
            .setSkipSignatureVerification()
            .build();

    //The first JwtConsumer is basically just used to parse the JWT into a JwtContext object.
    JwtContext jwtContext = firstPassJwtConsumer.process(jwt);

    // From the JwtContext we can get the issuer, or whatever else we might need,
    // to lookup or figure out the kind of validation policy to apply
    String issuer = jwtContext.getJwtClaims().getIssuer();

    // Just using the same key here but you might, for example, have a JWKS URIs configured for
    // each issuer, which you'd use to set up a HttpsJwksVerificationKeyResolver
    Key verificationKey = rsaJsonWebKey.getKey();

    // Using info from the JwtContext, this JwtConsumer is set up to verify
    // the signature and validate the claims.
    JwtConsumer secondPassJwtConsumer = new JwtConsumerBuilder()
            .setExpectedIssuer(issuer)
            .setVerificationKey(verificationKey)
            .setRequireExpirationTime()
            .setAllowedClockSkewInSeconds(30)
            .setRequireSubject()
            .setExpectedAudience("Audience")
            .build();

    // Finally using the second JwtConsumer to actually validate the JWT. This operates on
    // the JwtContext from the first processing pass, which avoids redundant parsing/processing.
    secondPassJwtConsumer.processContext(jwtContext);
```