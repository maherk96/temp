```java
package com.citi.fx.qa.qap.db.repos;

import com.citi.fx.qa.qap.db.entities.Application;
import com.citi.fx.qa.qap.db.entities.Tag;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.jdbc.core.JdbcTemplate;

import jakarta.persistence.EntityManager;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class TagRepositoryTest {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private EntityManager entityManager;

    @Autowired
    private TagRepository tagRepository;

    @BeforeEach
    void createSchema() {
        jdbcTemplate.execute("CREATE SCHEMA IF NOT EXISTS QAPORTAL");
    }

    @Test
    @DisplayName("Should find tag by name ignoring case")
    void testFindByNameIgnoreCase() {
        // given
        Application app = persistApp("TestApp1");

        Tag tag = new Tag();
        tag.setName("Regression");
        tag.setApp(app);
        tagRepository.save(tag);

        // when
        Optional<Tag> foundLower = tagRepository.findByNameIgnoreCase("regression");
        Optional<Tag> foundUpper = tagRepository.findByNameIgnoreCase("REGRESSION");

        // then
        assertThat(foundLower).isPresent();
        assertThat(foundUpper).isPresent();
        assertThat(foundLower.get().getName()).isEqualTo("Regression");
    }

    @Test
    @DisplayName("Should find all tags for given app id ordered by name ascending")
    void testFindByApp_IdOrderByNameAsc() {
        // given
        Application app = persistApp("TestApp2");

        Tag tagA = new Tag();
        tagA.setName("Alpha");
        tagA.setApp(app);

        Tag tagB = new Tag();
        tagB.setName("Beta");
        tagB.setApp(app);

        Tag tagC = new Tag();
        tagC.setName("Zeta");
        tagC.setApp(app);

        tagRepository.saveAll(List.of(tagC, tagA, tagB));

        // when
        List<Tag> tags = tagRepository.findByApp_IdOrderByNameAsc(app.getId());

        // then
        assertThat(tags).hasSize(3);
        assertThat(tags)
                .extracting(Tag::getName)
                .containsExactly("Alpha", "Beta", "Zeta");
    }

    private Application persistApp(String name) {
        Application app = new Application();
        app.setName(name);
        entityManager.persist(app);
        entityManager.flush();
        return app;
    }
}
```
