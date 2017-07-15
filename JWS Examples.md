[TOC]

## JWS Code Examples ##

### Signing with JWS ###
```
#!java
    //
    // An example of signing using JSON Web Signature (JWS)
    //

    // The content that will be signed
    String examplePayload = "This is some text that is to be signed.";

    // Create a new JsonWebSignature
    JsonWebSignature jws = new JsonWebSignature();

    // Set the payload, or signed content, on the JWS object
    jws.setPayload(examplePayload);

    // Set the signature algorithm on the JWS that will integrity protect the payload
    jws.setAlgorithmHeaderValue(AlgorithmIdentifiers.ECDSA_USING_P256_CURVE_AND_SHA256);

    // Set the signing key on the JWS
    // Note that your application will need to determine where/how to get the key
    // and here we just use an example from the JWS spec
    PrivateKey privateKey = ExampleEcKeysFromJws.PRIVATE_256;
    jws.setKey(privateKey);

    // Sign the JWS and produce the compact serialization or complete JWS representation, which
    // is a string consisting of three dot ('.') separated base64url-encoded
    // parts in the form Header.Payload.Signature
    String jwsCompactSerialization = jws.getCompactSerialization();

    // Do something useful with your JWS
    System.out.println(jwsCompactSerialization);

```

### JWS Verification Using a JWK ###


```
#!java

    //
    // An example of signature verification using JSON Web Signature (JWS)
    // where the verification key is obtained from a JSON Web Key Set document.
    //

    // A JSON Web Key (JWK) is a JavaScript Object Notation (JSON) data structure that represents a
    // cryptographic key (often but not always a public key). A JSON Web Key Set (JWK Set) document
    // is a JSON data structure for representing one or more JSON Web Keys (JWK). A JWK Set might,
    // for example, be obtained from an HTTPS endpoint controlled by the signer but this example
    // presumes the JWK Set JSONhas already been acquired by some secure/trusted means.
    String jsonWebKeySetJson = "{\"keys\":[" +
            "{\"kty\":\"EC\",\"use\":\"sig\"," +
             "\"kid\":\"the key\"," +
             "\"x\":\"amuk6RkDZi-48mKrzgBN_zUZ_9qupIwTZHJjM03qL-4\"," +
             "\"y\":\"ZOESj6_dpPiZZR-fJ-XVszQta28Cjgti7JudooQJ0co\",\"crv\":\"P-256\"}," +
            "{\"kty\":\"EC\",\"use\":\"sig\"," +
            " \"kid\":\"other key\"," +
             "\"x\":\"eCNZgiEHUpLaCNgYIcvWzfyBlzlaqEaWbt7RFJ4nIBA\"," +
             "\"y\":\"UujFME4pNk-nU4B9h4hsetIeSAzhy8DesBgWppiHKPM\",\"crv\":\"P-256\"}]}";

    // The complete JWS representation, or compact serialization, is string consisting of
    // three dot ('.') separated base64url-encoded parts in the form Header.Payload.Signature
    String compactSerialization = "eyJhbGciOiJFUzI1NiIsImtpZCI6InRoZSBrZXkifQ." +
            "UEFZTE9BRCE."+
            "Oq-H1lk5G0rl6oyNM3jR5S0-BZQgTlamIKMApq3RX8Hmh2d2XgB4scvsMzGvE-OlEmDY9Oy0YwNGArLpzXWyjw";

    // Create a new JsonWebSignature object
    JsonWebSignature jws = new JsonWebSignature();

    // Set the algorithm constraints based on what is agreed upon or expected from the sender
    jws.setAlgorithmConstraints(new AlgorithmConstraints(ConstraintType.WHITELIST,   AlgorithmIdentifiers.ECDSA_USING_P256_CURVE_AND_SHA256));

    // Set the compact serialization on the JWS
    jws.setCompactSerialization(compactSerialization);

    // Create a new JsonWebKeySet object with the JWK Set JSON
    JsonWebKeySet jsonWebKeySet = new JsonWebKeySet(jsonWebKeySetJson);

    // The JWS header contains information indicating which key was used to secure the JWS.
    // In this case (as will hopefully often be the case) the JWS Key ID
    // corresponds directly to the Key ID in the JWK Set.
    // The VerificationJwkSelector looks at Key ID, Key Type, designated use (signatures vs. encryption),
    // and the designated algorithm in order to select the appropriate key for verification from
    // a set of JWKs.
    VerificationJwkSelector jwkSelector = new VerificationJwkSelector();
    JsonWebKey jwk = jwkSelector.select(jws, jsonWebKeySet.getJsonWebKeys());

    // The verification key on the JWS is the public key from the JWK we pulled from the JWK Set.
    jws.setKey(jwk.getKey());

    // Check the signature
    boolean signatureVerified = jws.verifySignature();

    // Do something useful with the result of signature verification
    System.out.println("JWS Signature is valid: " + signatureVerified);

    // Get the payload, or signed content, from the JWS
    String payload = jws.getPayload();

    // Do something useful with the content
    System.out.println("JWS payload: " + payload);
```

### Signature Verification using JWS ###


```
#!java

    //
    // An example of signature verification using JSON Web Signature (JWS)
    //

    // The complete JWS representation, or compact serialization, is string consisting of
    // three dot ('.') separated base64url-encoded parts in the form Header.Payload.Signature
    String compactSerialization = "eyJhbGciOiJFUzI1NiJ9." +
            "VGhpcyBpcyBzb21lIHRleHQgdGhhdCBpcyB0byBiZSBzaWduZWQu." +
            "GHiNd8EgKa-2A4yJLHyLCqlwoSxwqv2rzGrvUTxczTYDBeUHUwQRB3P0dp_DALL0jQIDz2vQAT_cnWTIW98W_A";

    // Create a new JsonWebSignature
    JsonWebSignature jws = new JsonWebSignature();

    // Set the algorithm constraints based on what is agreed upon or expected from the sender
    jws.setAlgorithmConstraints(new AlgorithmConstraints(ConstraintType.WHITELIST, AlgorithmIdentifiers.ECDSA_USING_P256_CURVE_AND_SHA256));

    // Set the compact serialization on the JWS
    jws.setCompactSerialization(compactSerialization);

    // Set the verification key
    // Note that your application will need to determine where/how to get the key
    // Here we use an example from the JWS spec
    PublicKey publicKey = ExampleEcKeysFromJws.PUBLIC_256;
    jws.setKey(publicKey);

    // Check the signature
    boolean signatureVerified = jws.verifySignature();

    // Do something useful with the result of signature verification
    System.out.println("JWS Signature is valid: " + signatureVerified);

    // Get the payload, or signed content, from the JWS
    String payload = jws.getPayload();

    // Do something useful with the content
    System.out.println("JWS payload: " + payload);
```


### Using the RFC 7797 JWS Unencoded Payload Option ###


```
#!java

    //
    // An example of signature application and verification using the
    // RFC 7797 JSON Web Signature (JWS) Unencoded Payload Option with  
    // the "b64" header set to false indicating that the payload is not
    // base64url encoded when calculating the signature 
    //

    // The public/private key pair for this example as a JWK
    PublicJsonWebKey jwk = PublicJsonWebKey.Factory.newPublicJwk("{" +
            "  \"kty\": \"EC\"," +
            "  \"d\": \"Tk7qzHNnSBMioAU7NwZ9JugFWmWbUCyzeBRjVcTp_so\"," +
            "  \"use\": \"sig\"," +
            "  \"crv\": \"P-256\"," +
            "  \"kid\": \"example\"," +
            "  \"x\": \"qqeGjWmYZU5M5bBrRw1zqZcbPunoFVxsfaa9JdA0R5I\"," +
            "  \"y\": \"wnoj0YjheNP80XYh1SEvz1-wnKByEoHvb6KrDcjMuWc\"" +
            "}");

    // The message to be signed
    String content = "this is the content that will be signed";

    // Create a new JsonWebSignature object for the signing
    JsonWebSignature signerJws = new JsonWebSignature();

    // The content is the payload of the JWS
    signerJws.setPayload(content);

    // Set the signature algorithm on the JWS
    signerJws.setAlgorithmHeaderValue(AlgorithmIdentifiers.ECDSA_USING_P256_CURVE_AND_SHA256);

    // The private key is used to sign
    signerJws.setKey(jwk.getPrivateKey());

    // Set the Key ID (kid) header because it's just the polite thing to do.
    signerJws.setKeyIdHeaderValue(jwk.getKeyId());

    // Set the "b64" header to false, which indicates that the payload is not encoded
    // when calculating the signature (per RFC 7797)
    signerJws.getHeaders().setObjectHeaderValue(HeaderParameterNames.BASE64URL_ENCODE_PAYLOAD, false);

    // RFC 7797 requires that the b64" header be listed as critical
    signerJws.setCriticalHeaderNames(HeaderParameterNames.BASE64URL_ENCODE_PAYLOAD);

    // Produce the compact serialization with an empty/detached payload,
    // which is the encoded header + ".." + the encoded signature
    String detachedContentJws = signerJws.getDetachedContentCompactSerialization();

    // Send the JWS and content somewhere
    // The content might be the body of an HTTP message while the JWS value is carried in a header
    System.out.println(detachedContentJws);
    System.out.println(content);

    // Use a JsonWebSignature object to verify the signature
    JsonWebSignature verifierJws = new JsonWebSignature();

    // Set the algorithm constraints based on what is agreed upon or expected from the sender
    verifierJws.setAlgorithmConstraints(new AlgorithmConstraints(
            AlgorithmConstraints.ConstraintType.WHITELIST,
            AlgorithmIdentifiers.ECDSA_USING_P256_CURVE_AND_SHA256));

    // The JWS with detached content is the compact serialization
    verifierJws.setCompactSerialization(detachedContentJws);

    // The unencoded detached content is the payload
    verifierJws.setPayload(content);

    // The public key is used to verify the signature
    verifierJws.setKey(jwk.getPublicKey());

    // Check the signature
    boolean signatureVerified = verifierJws.verifySignature();

    // Do whatever needs to be done with the result of signature verification
    System.out.println("JWS Signature is valid: " + signatureVerified);
  
```