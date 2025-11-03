```java
package com.citi.fx.qa.qap.db.repos;

import com.citi.fx.qa.qap.db.entities.Application;
import com.citi.fx.qa.qap.db.entities.Tag;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class TagRepositoryTest {

    @Autowired
    private TagRepository tagRepository;

    @Autowired
    private ApplicationRepository applicationRepository; // only if you have it

    @Test
    @DisplayName("Should find tag by name ignoring case")
    void testFindByNameIgnoreCase() {
        // given
        Tag tag = new Tag();
        tag.setName("Regression");
        tag.setApp(createApp(100L));
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
    @DisplayName("Should return empty when tag name not found (case-insensitive)")
    void testFindByNameIgnoreCase_NotFound() {
        Tag tag = new Tag();
        tag.setName("Smoke");
        tag.setApp(createApp(200L));
        tagRepository.save(tag);

        Optional<Tag> result = tagRepository.findByNameIgnoreCase("Performance");

        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("Should find all tags for given appId ordered by name ascending")
    void testFindByApp_IdOrderByNameAsc() {
        // given
        Application app = createApp(300L);

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

    // helper method to create application
    private Application createApp(Long id) {
        Application app = new Application();
        app.setId(id); // If your Application has generated IDs, you can omit this line
        app.setName("TestApp-" + id);
        return app;
    }
}
```
