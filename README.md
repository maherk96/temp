```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import javax.persistence.EntityManagerFactory;
import javax.persistence.PersistenceUnit;
import javax.persistence.Table;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.DatabaseMetaData;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.HashSet;
import java.util.Set;
import java.util.stream.Collectors;

@SpringBootTest
class SchemaDiffTest {
    
    @PersistenceUnit
    private EntityManagerFactory emf;
    
    @Autowired
    private DataSource dataSource;
    
    @Test
    void findExtraTables() throws SQLException {
        // Get expected tables from JPA entities
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
        
        // Get actual tables from database
        Set<String> actualTables = new HashSet<>();
        try (Connection conn = dataSource.getConnection()) {
            DatabaseMetaData metaData = conn.getMetaData();
            try (ResultSet rs = metaData.getTables(null, "public", "%", new String[]{"TABLE"})) {
                while (rs.next()) {
                    actualTables.add(rs.getString("TABLE_NAME").toLowerCase());
                }
            }
        }
        
        // Calculate differences
        Set<String> extraTables = new HashSet<>(actualTables);
        extraTables.removeAll(expectedTables);
        
        Set<String> missingTables = new HashSet<>(expectedTables);
        missingTables.removeAll(actualTables);
        
        // Print results
        System.out.println("\n=== SCHEMA COMPARISON RESULTS ===\n");
        
        System.out.println("Expected tables from entities: " + expectedTables.size());
        System.out.println("Actual tables in database: " + actualTables.size());
        System.out.println();
        
        if (!extraTables.isEmpty()) {
            System.out.println("EXTRA TABLES (in database but no entity exists):");
            extraTables.forEach(t -> System.out.println("  ❌ " + t));
            System.out.println();
        }
        
        if (!missingTables.isEmpty()) {
            System.out.println("MISSING TABLES (entity exists but not in database):");
            missingTables.forEach(t -> System.out.println("  ⚠️  " + t));
            System.out.println();
        }
        
        if (extraTables.isEmpty() && missingTables.isEmpty()) {
            System.out.println("✅ All tables match perfectly!");
        }
        
        System.out.println("=================================\n");
    }
}
```
