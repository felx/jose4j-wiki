Recently (March '15) some [vulnerabilities were discovered/reported in some JWT/JWS libraries](https://auth0.com/blog/2015/03/31/critical-vulnerabilities-in-json-web-token-libraries/). Tim McLean, [the guy who did the research](https://www.timmclean.net/2015/03/31/jwt-algorithm-confusion.html), contacted me a few weeks ago with his early findings and suggested I look into things in the jose4j JOSE and JWT library. In the interest of transparency, and because there's some potentially useful info in it, I'm posting his email and my response and analysis here:


**On Wed, Mar 11, 2015 at 10:28 AM, Brian Campbell <...> wrote:**

Receipt acknowledged Tim,

I am the author/maintainer of https://bitbucket.org/b_c/jose4j and I'm pleased to say that the library is not vulnerable to, or has mitigation incorporated for, each of the potential exploits you've mentioned.

For (1), the library has an algorithm constraints concept https://bitbucket.org/b_c/jose4j/src/5f0869c5231e56cba68dc0a8a5f007a0648eca62/src/main/java/org/jose4j/jwa/AlgorithmConstraints.java?at=master#cl-30 which serves a similar purpose to the third parameter to specify the expected algorithm you suggest but is somewhat more versatile in that it can blacklist or whitelist an individual algorithm or set of algorithms. By default all JWS objects have algorithm constraints set to blacklist the "none" algorithm https://bitbucket.org/b_c/jose4j/src/5f0869c5231e56cba68dc0a8a5f007a0648eca62/src/main/java/org/jose4j/jws/JsonWebSignature.java?at=master#cl-35 so a library user won't unintentionally accept unsecured JWSs/JWTs. Furthermore, if a JWS object with the "none" algorithm is given a key, it will fail verification (the thinking being that if the library user gives a key, they are expected a real signed JWS).

For (2), in Java (and most statically typed languages, I'd guess) this doesn't work because the HMAC and public key verification algorithms expect different object types - public vs. symmetric key - and things just fail in the case you describe when the verifier uses a public key and the JWS/JWT has an HMAC algorithm. I did just write a few tests to verify that things do, in fact, work that way https://bitbucket.org/b_c/jose4j/src/5f0869c5231e56cba68dc0a8a5f007a0648eca62/src/test/java/org/jose4j/jws/PublicKeyAsHmacKeyTest.java?at=master but please do let me know if I've overlooked something. Also the algorithm constraints mentioned for (1) can also be used to blacklist all the HMAC algorithms in cases where only RSA or EC verification is expected. 

For (3), there's a time constant byte array comparison method https://bitbucket.org/b_c/jose4j/src/5f0869c5231e56cba68dc0a8a5f007a0648eca62/src/main/java/org/jose4j/lang/ByteUtil.java?at=master#cl-80 that is used by all the HMAC JWS algorithm implementations https://bitbucket.org/b_c/jose4j/src/5f0869c5231e56cba68dc0a8a5f007a0648eca62/src/main/java/org/jose4j/jws/HmacUsingShaAlgorithm.java?at=master#cl-45 as well as the tag check on the composite AES-CBC HMAC-SHA2 encryption algorithms https://bitbucket.org/b_c/jose4j/src/5f0869c5231e56cba68dc0a8a5f007a0648eca62/src/main/java/org/jose4j/jwe/AesCbcHmacSha2ContentEncryptionAlgorithm.java?at=master#cl-123

Hopefully that explains things sufficiently with respect to the jose4j JOSE and JWT library. But please let me know if I've overlooked something or further explanation or clarification is needed. 

Regards,
Brian 


**On Mon, Mar 9, 2015 at 9:36 PM, Tim McLean <...> wrote:**

Hello,

Through my research on the security of various implementations of JSON Web Tokens I found a number of vulnerabilities allowing attackers to bypass the verification step.  After reviewing a few libraries, the problem seems to be widespread across many languages and platforms.  Unfortunately, I don't have time to check every library myself, but I would appreciate it if you could check your implementation to see if it is vulnerable.  I believe similar problems may affect Java Web Signatures.

(1)
Applies if you:
- rely on and do not validate the `alg` field in the token header.
- implement the "none" algorithm.

Impact:
- Attackers can craft a malicious token containing an arbitrary payload that passes the verification step.

Exploit:
Create a token with the header `{"typ":"JWT","alg":"none"}`.  Include any payload.  Do not include a signature (i.e. the token should end with a period).
Note: some implementations include some basic but insufficient checking for a missing signature -- some minor fiddling may be required to produce an exploit.

(2)
Applies if you:
- rely on and do not validate the `alg` field in the token header.
- implement at least one of the HMAC algorithms and at least one of the asymmetric algorithms (e.g. HS256 and RS256).

Impact:
- If the system is expecting a token signed with one of the asymmetric algorithms, an attacker can bypass the verification step by knowing only the public key.

Exploit:
Create an HS256 token.  Generate the HMAC signature using the literal bytes of the public key file (often in the PEM format).  This will confuse the implementation into interpreting the public key file as an HMAC key.

Mitigation of (1) and (2):
Implementations should either validate the `alg` header field or ignore it completely.  Most libraries contain a `verify` or `decode` method that accepts two parameters: a token and a key.  I suggest adding a third parameter that requires the library user to specify the expected algorithm.  It is not safe to use the token's `alg` field as part of the verification step as the field is vulnerable to tampering (in fact, I believe this is a design flaw in JWT).

(3)
Applies if you:
- implement at least one of the HMAC algorithms
- do comparisons using `==` or your language's equivalent

Impact:
- Timing attack.  Attackers can measure how long to the signature verification takes to determine how many bytes of the signature they have guessed correctly.  This allows them to "brute force" the expected signature one byte at a time.  More information about timing attacks is available here.

Mitigation of (3):
When comparing the expected signature with the provided signature, a constant time comparison should be used.  Many languages provide a mechanism for this, but here is an example implementation.

I would appreciate if you could acknowledge receipt of this message so that I know I have reached a maintainer for each library.  I'm hoping to publish my findings March 18th -- let me know if this is too soon.

By the way, thanks for open sourcing your work! (that never really gets said enough)

Cheers,
Tim
