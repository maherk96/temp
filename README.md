```java

@SpringBootTest
class SchemaDiffTest {
    
    @PersistenceUnit
    private EntityManagerFactory emf;
    
    @Autowired
    private DataSource dataSource;
    
    @Test
    void findExtraTables() throws SQLException {
        Set<String> expectedTables = emf.getMetamodel().getEntities().stream()
            .map(entity -> {
                Class<?> javaType = entity.getJavaType();
                Table table = javaType.getAnnotation(Table.class);
                return (table != null && !table.name().isEmpty()) 
                    ? table.name() 
                    : entity.getName();
            })
            .map(String::toLowerCase)
            .collect(Collectors.toSet());
        
        Set<String> actualTables = new HashSet<>();
        try (Connection conn = dataSource.getConnection()) {
            DatabaseMetaData metaData = conn.getMetaData();
            try (ResultSet rs = metaData.getTables(null, "public", "%", new String[]{"TABLE"})) {
                while (rs.next()) {
                    actualTables.add(rs.getString("TABLE_NAME").toLowerCase());
                }
            }
        }
        
        Set<String> extraTables = new HashSet<>(actualTables);
        extraTables.removeAll(expectedTables);
        
        System.out.println("\nEXTRA TABLES:");
        extraTables.forEach(System.out::println);
    }
}

```
