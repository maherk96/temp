```yaml
plugins {
    id 'java-library'
    id "com.citi.171981.java.spring-boot"
    id 'org.liquibase.gradle' version '2.2.0'
}

configurations {
    all*.exclude module: 'spring-boot-starter-logging'
}

// Load database config from application.yml based on profile
import org.yaml.snakeyaml.Yaml

def loadYamlConfig(String profile) {
    def yaml = new Yaml()
    def baseConfig = [:]
    def profileConfig = [:]
    
    // Load base application.yml
    def baseFile = file("src/main/resources/application.yml")
    if (baseFile.exists()) {
        baseConfig = yaml.load(baseFile.text) ?: [:]
    }
    
    // Load profile-specific yml (e.g., application-dev.yml)
    if (profile) {
        def profileFile = file("src/main/resources/application-${profile}.yml")
        if (profileFile.exists()) {
            profileConfig = yaml.load(profileFile.text) ?: [:]
        }
    }
    
    return deepMerge(baseConfig, profileConfig)
}

def deepMerge(Map base, Map override) {
    def result = new LinkedHashMap(base)
    override.each { key, value ->
        if (value instanceof Map && result[key] instanceof Map) {
            result[key] = deepMerge(result[key], value)
        } else {
            result[key] = value
        }
    }
    return result
}

// Get profile from command line, default to 'dev'
def activeProfile = project.findProperty('profile') ?: 'dev'
def config = loadYamlConfig(activeProfile)

def dbUrl = config.spring?.datasource?.url ?: ''
def dbUsername = config.spring?.datasource?.username ?: ''
def dbPassword = config.spring?.datasource?.password ?: ''

dependencies {
    // Logging
    implementation "org.apache.logging.log4j:log4j-core:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-api:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-slf4j2-impl:${log4jVersion}"

    // Spring Boot Starters
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-validation"
    implementation "org.springframework.boot:spring-boot-starter-data-jpa"
    implementation 'org.springframework:spring-core:6.2.11'

    // MapStruct
    implementation "org.mapstruct:mapstruct:${mapstruct}"
    annotationProcessor "org.mapstruct:mapstruct-processor:${mapstruct}"

    // Database
    implementation "com.oracle.database.jdbc:ojdbc8:${jdbcVersion}"
    implementation "com.oracle.database.jdbc:ucp:${jdbcVersion}"
    implementation "com.h2database:h2:${h2Version}"

    // Liquibase
    implementation 'org.liquibase:liquibase-core'

    // Caching
    implementation "com.github.ben-manes.caffeine:caffeine:${caffeineVersion}"
    implementation "org.springframework.boot:spring-boot-starter-cache"

    // Messaging
    implementation "com.solace:solace-messaging-client:${solaceVersion}"

    // OpenAPI
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:${springdocVersion}"

    // Test Dependencies
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testImplementation "com.h2database:h2:${h2Version}"

    // Internal Dependencies
    implementation "com.citi.179300.fx-qap:qap-common:${qapCommon}"
    implementation "com.citi.179300.fx-qap:qap-logging-core:${qapLoggingCore}"
    testImplementation "com.citi.179300.fx-qap:qap-data-publisher:${qapDataPublisher}"
    testImplementation "com.citi.fx-qa-utils.qa-middleware-messaging:solace:${qaMiddlewareMsg}"
    testImplementation "com.citi.179300.fx-qa-utils:qa-awaitility:${qaAwaitility}"
    implementation "com.citi.fx-qa-utils:qa-kdb:${qaKdbVersion}"

    //override jackson to use non vulnerable versions
    implementation "com.fasterxml.jackson.core:jackson-core:${jacksonVersion}"

    // Embedded Tomcat
    implementation "org.apache.tomcat.embed:tomcat-embed-core:${tomcatVersion}"
    implementation "org.apache.tomcat.embed:tomcat-embed-websocket:${tomcatVersion}"
    implementation "org.apache.tomcat.embed:tomcat-embed-el:${tomcatVersion}"
    implementation 'org.springframework:spring-websocket'
    implementation 'org.springframework:spring-messaging'

    // Liquibase runtime dependencies for Gradle tasks
    liquibaseRuntime 'org.liquibase:liquibase-core:4.26.0'
    liquibaseRuntime 'org.liquibase.ext:liquibase-hibernate6:4.26.0'
    liquibaseRuntime 'org.springframework.boot:spring-boot-starter-data-jpa'
    liquibaseRuntime "com.oracle.database.jdbc:ojdbc8:${jdbcVersion}"
    liquibaseRuntime 'info.picocli:picocli:4.7.5'
    liquibaseRuntime 'javax.xml.bind:jaxb-api:2.3.1'
    liquibaseRuntime sourceSets.main.output
}

// Liquibase configuration - reads from application.yml
liquibase {
    activities {
        main {
            changelogFile 'src/main/resources/db/changelog/db.changelog-master.yaml'
            
            // Database connection from application.yml
            url dbUrl
            username dbUsername
            password dbPassword
            driver 'oracle.jdbc.OracleDriver'
            
            // Reference URL - points to your Hibernate entities
            referenceUrl 'hibernate:spring:com.citi.fx.qa.qap.db.domain?dialect=org.hibernate.dialect.OracleDialect'
            
            // Output file for generated changelogs
            outputChangelogFile 'src/main/resources/db/changelog/generated/output.yaml'
        }
    }
    runList = 'main'
}

// Ensure classes are compiled before Liquibase tasks run
tasks.matching { it.name.startsWith('liquibase') }.configureEach {
    dependsOn tasks.named('classes')
}

tasks.withType(JavaCompile) {
    options.compilerArgs += ["-parameters"]
}

test {
    useJUnitPlatform()
}
```
