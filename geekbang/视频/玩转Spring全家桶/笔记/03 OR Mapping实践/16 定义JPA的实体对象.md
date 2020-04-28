1. @MappedSuperclass
 - 标注该注解的类不是一个完整的实体类，不会映射到数据库表，其属性映射到子类的数据库字段中
 - 标注该注解的类不能再标注@Entity或@Table，也无需实现序列化接口

2. @Id、@Generator使用序列生成主键案例
 ```
@Entity(name = "Product")
@Data
public class Product {
    @Id
    @GeneratedValue(
            strategy = GenerationType.SEQUENCE,
            generator = "sequence-generator"
    )
    @SequenceGenerator(
            name = "sequence-generator",
            sequenceName = "product_sequence"
    )
    private Long id;
    @Column(name = "product_name")
    private String name;
}
 ```