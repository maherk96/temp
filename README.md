```java
private void createResponse(
    CreditCheckRequest request,
    boolean isPass,
    ResponseType type
) {
    creditCheckResponse.setSessionName(request.getSessionName());
    creditCheckResponse.setHeader(request.getHeader());
    creditCheckResponse.setProduct(request.getProduct());

    switch (type) {
        case CMP -> {
            creditCheckResponse.setCheckOk(isPass);

            String exception = isPass
                ? "ok CMPFlag CM. utilization " + request.getUtilization() +
                  ". limit " + request.getLimit() +
                  ". availability " + request.getAvailability()
                : "CMP check failed."; // adjust for fail if needed

            creditCheckResponse.setException(exception);
            creditCheckResponse.setExceptionType("NONE");

            String gfcid = request.getGfcid() != null ? request.getGfcid() : "0";
            creditCheckResponse.setGfcid(gfcid);

            // CMP does NOT use facilityId
        }
        case FACILITY -> {
            creditCheckResponse.setCheckOk(isPass);

            String exception;
            String exceptionType;

            if (isPass) {
                exception = "OK";
                exceptionType = "NONE";
            } else {
                exception = "PSE usage exceeds availability. Requested " +
                        request.getRequested() + ". Available " + request.getAvailable();
                exceptionType = "PSE";
            }

            creditCheckResponse.setException(exception);
            creditCheckResponse.setExceptionType(exceptionType);

            String facilityId = request.getFacilityId() != null ? request.getFacilityId() : "0";
            creditCheckResponse.setFacilityId(facilityId);

            String gfcid = request.getGfcid() != null ? request.getGfcid() : "0";
            creditCheckResponse.setGfcid(gfcid);
        }
        default -> throw new IllegalArgumentException("Unsupported ResponseType");
    }
}

```
