```java
package com.citi.fxpg.service;

import com.citi.fxpg.model.TradeBookingRequest;
import jakarta.xml.bind.JAXBContext;
import jakarta.xml.bind.JAXBException;
import jakarta.xml.bind.Unmarshaller;
import lombok.extern.slf4j.Slf4j;
import java.io.StringReader;
import java.io.InputStream;
import java.io.File;

/**
 * Unmarshaller for TradeBookingRequest XML messages
 */
@Slf4j
public class TradeBookingUnmarshaller {
    
    private final JAXBContext jaxbContext;
    private final Unmarshaller unmarshaller;
    
    public TradeBookingUnmarshaller() throws JAXBException {
        this.jaxbContext = JAXBContext.newInstance(TradeBookingRequest.class);
        this.unmarshaller = jaxbContext.createUnmarshaller();
        log.info("TradeBookingUnmarshaller initialized successfully");
    }
    
    /**
     * Unmarshal from XML string
     * @param xmlString the XML content as string
     * @return TradeBookingRequest object
     * @throws JAXBException if unmarshalling fails
     */
    public TradeBookingRequest unmarshalFromString(String xmlString) throws JAXBException {
        log.debug("Unmarshalling from XML string");
        StringReader reader = new StringReader(xmlString);
        TradeBookingRequest result = (TradeBookingRequest) unmarshaller.unmarshal(reader);
        log.debug("Successfully unmarshalled TradeBookingRequest");
        return result;
    }
    
    /**
     * Unmarshal from InputStream
     * @param inputStream the XML content as InputStream
     * @return TradeBookingRequest object
     * @throws JAXBException if unmarshalling fails
     */
    public TradeBookingRequest unmarshalFromStream(InputStream inputStream) throws JAXBException {
        log.debug("Unmarshalling from InputStream");
        TradeBookingRequest result = (TradeBookingRequest) unmarshaller.unmarshal(inputStream);
        log.debug("Successfully unmarshalled TradeBookingRequest from stream");
        return result;
    }
    
    /**
     * Unmarshal from File
     * @param file the XML file
     * @return TradeBookingRequest object
     * @throws JAXBException if unmarshalling fails
     */
    public TradeBookingRequest unmarshalFromFile(File file) throws JAXBException {
        log.debug("Unmarshalling from file: {}", file.getAbsolutePath());
        TradeBookingRequest result = (TradeBookingRequest) unmarshaller.unmarshal(file);
        log.debug("Successfully unmarshalled TradeBookingRequest from file");
        return result;
    }
}

```
