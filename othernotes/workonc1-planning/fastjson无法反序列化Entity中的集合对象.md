1. 先简单展示下自己的Entity如下，注意类PilotFormRowCell有个有参构造函数
```
public class PilotForm {
    ······
    private List<PilotFormRow> formRowList;
```
```
public class PilotFormRow {
    //@JSONField(deserializeUsing = CustomeDeserialize.class)
    private List<PilotFormRowCell> cellList;
```
```
public class PilotFormRowCell {
    private Integer rowIndex;
    private Integer colIndex;
······
    @JSONField(serialize = false)
    private PilotFormRow currRow;
······
    public PilotFormRowCell(PilotFormRow formRow) {
        this.currRow = formRow;
    }
```
2. 如上的一个Entity，使用fastjosn序列化后，再用方法```JSONObject.parseObject(new String(value),PilotForm.class); ```反序列化的时候，cellList始终是空对象，也没有报错。
3. 然后自己是写了一个反序列化类，打算自己处理数据返回。如下，在创建PilotFormRowCell对象时，发现没有无参构造器，然后就加了无参构造器，在不指定cellList使用特定反序列化类的情况下，cellList有对应值了。意识到反序列化问题是由于没有**无参构造器**导致
```
public class CustomeDeserialize extends AbstractDateDeserializer {
    @Override
    protected <T> T cast(DefaultJSONParser parser, Type clazz, Object fieldName, Object value) {
        List<PilotFormRowCell> cellList=new ArrayList<>();
        PilotFormRowCell cell=new PilotFormRowCell();
        cell.setCellContent("测试");
        cellList.add(cell);
        return (T) cellList;
    }
    @Override
    public int getFastMatchToken() {
        return JSONToken.LITERAL_INT;
    }
}
```