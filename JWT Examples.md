## JSON Web Token (JWT) Code Examples ##

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


    // Now you can something with the JWT. Like send it to some other party
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


    // There's also a key resolver that selects from among a given list of JWKs using the Key ID
    // and other factors provided in the header of the JWS/JWT.
    JsonWebKeySet jsonWebKeySet = new JsonWebKeySet(rsaJsonWebKey);
    JwksVerificationKeyResolver jwksResolver = new JwksVerificationKeyResolver(jsonWebKeySet.getJsonWebKeys());
    jwtConsumer = new JwtConsumerBuilder()
            // ... other set up of the JwtConsumerBuilder ...
            .setVerificationKeyResolver(jwksResolver)
            // ...
            .build();


    // Sometimes X509 certificate(s) are provided out-of-band somehow by the signer/issuer
    // and the X509VerificationKeyResolver is helpful for that situation. It will use
    // the X.509 Certificate Thumbprint Headers (x5t or x5t#S256) from the JWS/JWT to
    // select from among the provided certificates to get the public key for verification.
    List<X509Certificate> certificates = new ArrayList<>(); // add certs to this list
    X509VerificationKeyResolver x509VerificationKeyResolver = new X509VerificationKeyResolver(certificates);

    // Optionally the X509VerificationKeyResolver can attempt to verify the signature
    // with the key from each of the provided certificates, if no X.509 Certificate
    // Thumbprint Header is present in the JWT/JWS.
    x509VerificationKeyResolver.setTryAllOnNoThumbHeader(true);

    jwtConsumer = new JwtConsumerBuilder()
            // ... other set up of the JwtConsumerBuilder ...
            .setVerificationKeyResolver(x509VerificationKeyResolver)
            // ...
            .build();

```