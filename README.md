```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=Oracle;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: update  # or validate / create-drop — pick what suits your test style
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
  h2:
    console:
      enabled: true
      path: /h2-console

# Optional logging tweaks for debugging
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

# Optional: If you use Flyway or Liquibase
flyway:
  enabled: true
  locations: classpath:db/migration

# Profiles
spring:
  profiles:
    active: test

```
