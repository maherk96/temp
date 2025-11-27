```java
@Component
public class ProcessingGuard {

    @Value("${processor.hostname-guard-enabled:false}")
    private boolean guardEnabled;

    @Value("${processor.allowed-hosts:}")
    private List<String> allowedHosts;

    private String hostname;

    @PostConstruct
    public void init() throws Exception {
        this.hostname = InetAddress.getLocalHost().getHostName();
    }

    public boolean allowProcessing() {

        // Guard is OFF → always allow processing
        if (!guardEnabled) {
            return true;
        }

        // Guard is ON → only allow if hostname is in list
        return allowedHosts.contains(hostname);
    }

    public String getHostname() {
        return hostname;
    }
}


```
