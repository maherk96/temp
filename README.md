```java
@Entity
@Table(
    name = "test",
    uniqueConstraints = {
        @UniqueConstraint(columnNames = {"methodName", "displayName", "test_class_id"}),
        @UniqueConstraint(columnNames = {"methodName", "test_class_id"}),
        @UniqueConstraint(columnNames = {"test_feature_id"})
    }
)
@Getter
@Setter
public class Test {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Column(length = 100, nullable = false)
    private String methodName;

    @Column(length = 300)
    private String displayName;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "test_class_id", nullable = false)
    private TestClass testClass;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "test_feature_id", nullable = false)
    private TestFeature testFeature;

    @CreationTimestamp
    private Instant created;

    // other mappings ...
}

```
