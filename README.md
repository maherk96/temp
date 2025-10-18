```java
/**
 * Reads FIX messages from a classpath resource file
 * 
 * @param resourcePath path to the resource file (relative to classpath root)
 *                     e.g., "fix_messages.txt" for src/test/resources/fix_messages.txt
 * @return list of parsed FIX messages
 * @throws RuntimeException if resource cannot be read or parsed
 */
public static List<Message> readFixMessageFromFile(String resourcePath) {
    List<Message> messages = new ArrayList<>();
    
    try (InputStream is = FixUtil.class.getClassLoader().getResourceAsStream(resourcePath)) {
        if (is == null) {
            throw new IOException("Resource not found on classpath: " + resourcePath);
        }
        
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(is))) {
            String line;
            int lineNumber = 0;
            
            while ((line = reader.readLine()) != null) {
                lineNumber++;
                line = line.trim();
                
                if (line.isEmpty()) {
                    continue;
                }
                
                try {
                    log.info("Parsing FIX message: {}", line);
                    Message fixMessage = new Message(line, false);
                    messages.add(fixMessage);
                } catch (Exception e) {
                    String errorMsg = String.format(
                        "Error parsing FIX message at line %d: %s", 
                        lineNumber, 
                        line
                    );
                    log.error(errorMsg, e);
                    throw new RuntimeException(errorMsg, e);
                }
            }
        }
        
    } catch (IOException e) {
        String errorMsg = "Error reading FIX messages from resource: " + resourcePath;
        log.error(errorMsg, e);
        throw new RuntimeException(errorMsg, e);
    }
    
    return messages;
}

// Example of how to call this method:
public static void main(String[] args) {
    try {
        // CORRECT: Use classpath-relative path (no src/test/resources/ prefix)
        List<Message> messages = readFixMessageFromFile("fix_messages.txt");
        
        messages.forEach(message -> {
            System.out.println(message.toString());
        });
        
    } catch (RuntimeException e) {
        System.err.println("Failed to read messages: " + e.getMessage());
    }
}

/**
 * Alternative: Read from absolute file path (not classpath)
 * Use this if you need to read from file system directly
 */
public static List<Message> readFixMessageFromFilePath(Path filePath) {
    List<Message> messages = new ArrayList<>();
    
    try (BufferedReader reader = Files.newBufferedReader(filePath)) {
        String line;
        int lineNumber = 0;
        
        while ((line = reader.readLine()) != null) {
            lineNumber++;
            line = line.trim();
            
            if (line.isEmpty()) {
                continue;
            }
            
            try {
                log.info("Parsing FIX message: {}", line);
                Message fixMessage = new Message(line, false);
                messages.add(fixMessage);
            } catch (Exception e) {
                String errorMsg = String.format(
                    "Error parsing FIX message at line %d: %s", 
                    lineNumber, 
                    line
                );
                log.error(errorMsg, e);
                throw new RuntimeException(errorMsg, e);
            }
        }
        
    } catch (IOException e) {
        String errorMsg = "Error reading FIX messages from file: " + filePath;
        log.error(errorMsg, e);
        throw new RuntimeException(errorMsg, e);
    }
    
    return messages;
}

// Example using file path:
public static void example2() {
    try {
        Path filePath = Paths.get("src/test/resources/fix_messages.txt");
        List<Message> messages = readFixMessageFromFilePath(filePath);
        messages.forEach(System.out::println);
    } catch (RuntimeException e) {
        System.err.println("Failed to read messages: " + e.getMessage());
    }
}
```
