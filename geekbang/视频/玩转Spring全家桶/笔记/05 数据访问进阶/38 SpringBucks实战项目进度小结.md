1. 如果使用的是jackson序列化插件，可以通过实现StdSerializer<T>和StdDeserializer<T>接口对执行类型进行定制化的序列化和反序列化。然后在对应类型的字段上使用注解@JsonSerialize和@JsonDeserialize指定
 - Money类型序列化
```
public class MoneySerializer extends StdSerializer<Money> {
    protected MoneySerializer() {
        super(Money.class);
    }

    @Override
    public void serialize(Money money, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
        jsonGenerator.writeNumber(money.getAmountMinorLong());
    }
}
```
 -Money类型反序列化
```
public class MoneyDeserializer extends StdDeserializer<Money> {
    protected MoneyDeserializer() {
        super(Money.class);
    }

    @Override
    public Money deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        return Money.ofMinor(CurrencyUnit.of("CNY"), p.getLongValue());
    }
}
```
 - 类字段中使用方式
```
    @JsonSerialize(using = MoneySerializer.class)
    @JsonDeserialize(using = MoneyDeserializer.class)
    private Money price;
```