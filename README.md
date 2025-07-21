```kts
allprojects {
    configurations.all {
        resolutionStrategy {
            eachDependency { DependencyResolveDetails details ->
                if (details.requested.group == 'org.apache.commons' && details.requested.name == 'commons-lang3') {
                    details.useVersion '3.18.0'
                }
            }
        }
    }

    dependencies {
        constraints {
            implementation('org.apache.commons:commons-lang3:3.18.0') {
                because 'Fixes CVE-2025-48924'
            }
        }
    }
}
```
