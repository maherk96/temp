```java
private SSLContext sslContext() {
    try {
        X509ExtendedTrustManager trustAll = new X509ExtendedTrustManager() {
            @Override
            public void checkClientTrusted(X509Certificate[] chain, String authType, Socket socket) {}
            @Override
            public void checkServerTrusted(X509Certificate[] chain, String authType, Socket socket) {}
            @Override
            public void checkClientTrusted(X509Certificate[] chain, String authType, SSLEngine engine) {}
            @Override
            public void checkServerTrusted(X509Certificate[] chain, String authType, SSLEngine engine) {}
            @Override
            public void checkClientTrusted(X509Certificate[] chain, String authType) {}
            @Override
            public void checkServerTrusted(X509Certificate[] chain, String authType) {}
            @Override
            public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
        };

        SSLContext sc = SSLContext.getInstance("TLS");
        sc.init(null, new TrustManager[]{ trustAll }, new SecureRandom());
        return sc;
    } catch (Exception e) {
        throw new RuntimeException("Failed to create unsafe SSLContext", e);
    }
}
```
