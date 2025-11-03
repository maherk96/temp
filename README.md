```java
@Entity
@Table(name = "TEST_TAG", schema = "QAPORTAL")
@Getter
@Setter
public class TestTag {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "test_tag_seq")
    @SequenceGenerator(name = "test_tag_seq", sequenceName = "TEST_TAG_SEQ", allocationSize = 1)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TAG_ID")
    private Tag tag;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEST_ID")
    private Test test;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEST_LAUNCH_ID")
    private TestLaunch testLaunch;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant created;
}

@Entity
@Table(
    name = "TAG",
    schema = "QAPORTAL",
    uniqueConstraints = @UniqueConstraint(columnNames = {"APP_ID", "NAME"})
)
@Getter
@Setter
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 50, nullable = false)
    private String name;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "APP_ID", nullable = false)
    private Application app;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant created;
}

```
