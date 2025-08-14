```java
package com.example.restexec.api.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;

import java.util.List;
import java.util.Map;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class RestExecSpec {

    private String version;
    private String baseUrl;
    private Globals globals;
    private List<Request> requests;
    private Flow flow;
    private Load load;

    // Getters & Setters
    public String getVersion() { return version; }
    public void setVersion(String version) { this.version = version; }

    public String getBaseUrl() { return baseUrl; }
    public void setBaseUrl(String baseUrl) { this.baseUrl = baseUrl; }

    public Globals getGlobals() { return globals; }
    public void setGlobals(Globals globals) { this.globals = globals; }

    public List<Request> getRequests() { return requests; }
    public void setRequests(List<Request> requests) { this.requests = requests; }

    public Flow getFlow() { return flow; }
    public void setFlow(Flow flow) { this.flow = flow; }

    public Load getLoad() { return load; }
    public void setLoad(Load load) { this.load = load; }

    // ---------- Nested DTOs ----------

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Globals {
        private Map<String, String> headers;
        private Map<String, String> vars;
        private Retry retry;
        private Timeouts timeouts;

        public Map<String, String> getHeaders() { return headers; }
        public void setHeaders(Map<String, String> headers) { this.headers = headers; }

        public Map<String, String> getVars() { return vars; }
        public void setVars(Map<String, String> vars) { this.vars = vars; }

        public Retry getRetry() { return retry; }
        public void setRetry(Retry retry) { this.retry = retry; }

        public Timeouts getTimeouts() { return timeouts; }
        public void setTimeouts(Timeouts timeouts) { this.timeouts = timeouts; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Retry {
        private int maxAttempts;
        private int backoffMs;
        private boolean jitter;
        private List<Integer> retryOn;

        public int getMaxAttempts() { return maxAttempts; }
        public void setMaxAttempts(int maxAttempts) { this.maxAttempts = maxAttempts; }

        public int getBackoffMs() { return backoffMs; }
        public void setBackoffMs(int backoffMs) { this.backoffMs = backoffMs; }

        public boolean isJitter() { return jitter; }
        public void setJitter(boolean jitter) { this.jitter = jitter; }

        public List<Integer> getRetryOn() { return retryOn; }
        public void setRetryOn(List<Integer> retryOn) { this.retryOn = retryOn; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Timeouts {
        private int connectMs;
        private int readMs;

        public int getConnectMs() { return connectMs; }
        public void setConnectMs(int connectMs) { this.connectMs = connectMs; }

        public int getReadMs() { return readMs; }
        public void setReadMs(int readMs) { this.readMs = readMs; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Request {
        private String name;
        private String method;
        private String path;
        private Map<String, String> query;
        private Map<String, Object> body;
        private Map<String, String> save;
        private Assert assertSection;

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }

        public String getMethod() { return method; }
        public void setMethod(String method) { this.method = method; }

        public String getPath() { return path; }
        public void setPath(String path) { this.path = path; }

        public Map<String, String> getQuery() { return query; }
        public void setQuery(Map<String, String> query) { this.query = query; }

        public Map<String, Object> getBody() { return body; }
        public void setBody(Map<String, Object> body) { this.body = body; }

        public Map<String, String> getSave() { return save; }
        public void setSave(Map<String, String> save) { this.save = save; }

        @JsonProperty("assert")
        public Assert getAssertSection() { return assertSection; }
        public void setAssertSection(Assert assertSection) { this.assertSection = assertSection; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Assert {
        private int status;
        private Map<String, String> jsonPath;

        public int getStatus() { return status; }
        public void setStatus(int status) { this.status = status; }

        public Map<String, String> getJsonPath() { return jsonPath; }
        public void setJsonPath(Map<String, String> jsonPath) { this.jsonPath = jsonPath; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Flow {
        private Map<String, String> variables;
        private ThinkTime thinkTimeMs;
        private String onError;

        public Map<String, String> getVariables() { return variables; }
        public void setVariables(Map<String, String> variables) { this.variables = variables; }

        public ThinkTime getThinkTimeMs() { return thinkTimeMs; }
        public void setThinkTimeMs(ThinkTime thinkTimeMs) { this.thinkTimeMs = thinkTimeMs; }

        public String getOnError() { return onError; }
        public void setOnError(String onError) { this.onError = onError; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class ThinkTime {
        private int min;
        private int max;

        public int getMin() { return min; }
        public void setMin(int min) { this.min = min; }

        public int getMax() { return max; }
        public void setMax(int max) { this.max = max; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Load {
        private String model;
        private int users;
        private String rampUp;
        private String holdFor;
        private int iterations;
        private String warmup;
        private StopOn stopOn;

        public String getModel() { return model; }
        public void setModel(String model) { this.model = model; }

        public int getUsers() { return users; }
        public void setUsers(int users) { this.users = users; }

        public String getRampUp() { return rampUp; }
        public void setRampUp(String rampUp) { this.rampUp = rampUp; }

        public String getHoldFor() { return holdFor; }
        public void setHoldFor(String holdFor) { this.holdFor = holdFor; }

        public int getIterations() { return iterations; }
        public void setIterations(int iterations) { this.iterations = iterations; }

        public String getWarmup() { return warmup; }
        public void setWarmup(String warmup) { this.warmup = warmup; }

        public StopOn getStopOn() { return stopOn; }
        public void setStopOn(StopOn stopOn) { this.stopOn = stopOn; }
    }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class StopOn {
        private double errorRatePct;
        private int p95LtMs;

        public double getErrorRatePct() { return errorRatePct; }
        public void setErrorRatePct(double errorRatePct) { this.errorRatePct = errorRatePct; }

        public int getP95LtMs() { return p95LtMs; }
        public void setP95LtMs(int p95LtMs) { this.p95LtMs = p95LtMs; }
    }
}

```
