```java
package com.example.fixutils;

import java.security.SecureRandom;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class FixOrderIdGenerator {

    private static final String ALPHA_NUMERIC = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    private static final SecureRandom RANDOM = new SecureRandom();

    /**
     * Generates a random ClOrdID in the format:
     * O<random><timestamp><random>
     * Example: O7SCDDT00069H1P
     */
    public static String generateClOrdID() {
        String prefix = "O";
        String random1 = randomString(5);
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("HHmmss"));
        String random2 = randomString(3);
        return prefix + random1 + timestamp + random2;
    }

    /**
     * Generates a random ClOrdLinkID in the format:
     * CSU<random><timestamp>
     * Example: CSU1V35MA00069
     */
    public static String generateClOrdLinkID() {
        String prefix = "CSU";
        String random = randomString(5);
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("mmssSS"));
        return prefix + random + timestamp;
    }

    /** Utility to create a random alphanumeric string of given length */
    private static String randomString(int length) {
        StringBuilder sb = new StringBuilder(length);
        for (int i = 0; i < length; i++) {
            sb.append(ALPHA_NUMERIC.charAt(RANDOM.nextInt(ALPHA_NUMERIC.length())));
        }
        return sb.toString();
    }

    // Example usage
    public static void main(String[] args) {
        System.out.println("ClOrdID     " + generateClOrdID());
        System.out.println("ClOrdLinkID " + generateClOrdLinkID());
    }
}


```
