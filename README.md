```java
package com.yourcompany.qaportal.dto;

import jakarta.validation.constraints.Size;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Data
@NoArgsConstructor
public class TagDTO {

    private Long id;

    @Size(max = 50)
    private String name;

    private Long app;

    private String created;

    private TagDTO(String name, Long app) {
        this.name = name;
        this.app = app;
    }

    public static TagDTO createTag(String name, Long app) {
        log.debug("Creating tag '{}' for appId {}", name, app);
        return new TagDTO(name, app);
    }
}

```
