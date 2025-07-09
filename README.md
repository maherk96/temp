```java
public class CreditCheckClientConnection {

    private final String srcSystem;
    private final String configFile;
    private final String primingAccn;

    private CreditCheckClient creditCheckClient;
    private CreditCheckClientSession session;

    public CreditCheckClientConnection(String srcSystem, String configFile, String primingAccn) {
        this.srcSystem = srcSystem;
        this.configFile = configFile;
        this.primingAccn = primingAccn;

        init();
    }

    private void init() {
        try {
            this.creditCheckClient = CreditCheckClientFactory.getInstance(configFile);
            this.creditCheckClient.start();
            this.session = this.creditCheckClient.createSession();
            // TODO: prime session with primingAccn if needed
        } catch (Exception e) {
            throw new RuntimeException("Failed to init CreditCheckClient", e);
        }
    }

    public boolean isConnected() {
        return creditCheckClient != null && session != null;
    }

    public void retry() {
        if (creditCheckClient != null) {
            creditCheckClient.stop();
            this.session = creditCheckClient.createSession();
        }
    }

    public CreditCheckResponse send(FxCreditCheckRequest request) {
        String requestId = session.sendCreditCheckRequest(request);
        return session.receiveCreditCheckResponse(requestId);
    }
}
```
