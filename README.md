```java
import lombok.Data;

@Data
public class Header {
    private String cliVer;
    private String hostname;
    private String pid;
    private String msgType;
    private String product;
    private String userId;
    private String baseNum;
    private String bvShortName;
    private String requestType;
    private String earmarkType;
    private String sourceSystem;
    private boolean fastRequest;
    private String correlationId;
}

import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import lombok.Data;

@Data
public class CreditCheckResponse {

    private String sessionName;

    @JsonDeserialize(using = NestedJsonStringDeserializer.class)
    private Header header;

    private boolean checkOk;
    private String exception;
    private String exceptionType;
    private String product;
    private String facilityId;
    private String gfcid;
    private String messageType;

    // Add other fields as needed.
}

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.IOException;

public class NestedJsonStringDeserializer extends JsonDeserializer<Header> {

    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public Header deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String jsonString = p.getValueAsString();
        return mapper.readValue(jsonString, Header.class);
    }
}


```
