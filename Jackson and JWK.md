The code snippets below show how a custom Jackson Json serializer and deserializer can be used to properly serialize/deserialize JsonWebKey objects that are part of a larger object tree.     

```
#!java

public class JWKJsonDeserializer extends StdDeserializer<JsonWebKey> {
    public JWKJsonDeserializer() {
        super(JsonWebKey.class);
    }

    @Override
    public JsonWebKey deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
        final JsonLocation currentLocation = jsonParser.getCurrentLocation();
        try {
            return JsonWebKey.Factory.newJwk(jsonParser.readValueAs(Map.class));
        } catch (JoseException e) {
            throw new JsonParseException("Unable to parse Json Web Key", currentLocation, e);
        }
    }
}
```

```
#!java

public class JWKJsonSerializer extends StdSerializer<JsonWebKey> {
    public JWKJsonSerializer() {
        super(JsonWebKey.class);
    }

    @Override
    public void serialize(JsonWebKey jsonWebKey, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException, JsonGenerationException {
        jsonGenerator.writeObject(jsonWebKey.toParams(OutputControlLevel.INCLUDE_SYMMETRIC));
    }
}
```

```
#!java

// Add the serializer and deserializer to the field containing the key.
public static class TestObject {
    @JsonSerialize(using = JWKJsonSerializer.class)
    @JsonDeserialize(using = JWKJsonDeserializer.class)
    public JsonWebKey key;
}
```
