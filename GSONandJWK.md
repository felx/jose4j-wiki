


```
#!java

public class JwkDeserializer implements JsonDeserializer<JsonWebKey>
{
    public JsonWebKey deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context)
            throws JsonParseException
    {
        Map<String,Object> jwkParameters = context.deserialize(json, LinkedHashMap.class);
        try
        {
            return JsonWebKey.Factory.newJwk(jwkParameters);
        }
        catch (JoseException e)
        {
            throw new JsonParseException("Unable to create JWK Object when parsing JSON", e);
        }
    }
}

```


```
#!java

public class JwkSerializer implements JsonSerializer<JsonWebKey>
{
    public JsonElement serialize(JsonWebKey src, Type typeOfSrc, JsonSerializationContext context)
    {
        Map<String, Object> jwkParameters = src.toParams(JsonWebKey.OutputControlLevel.PUBLIC_ONLY);
        return context.serialize(jwkParameters);
    }
}

```



```
#!java

    // register our own custom serializer and deserializer for JsonWebKey objects, 
    // which deal with the JWK using a map to represent the JSON parameters of the JWK
    Gson gson = new GsonBuilder()
            .registerTypeAdapter(JsonWebKey.class, new JwkDeserializer())
            .registerTypeAdapter(JsonWebKey.class, new JwkSerializer())
            .setPrettyPrinting().create();
```