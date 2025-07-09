```java
import java.util.concurrent.ThreadLocalRandom;

public class CustomIdGenerator {

    // Hardcoded parts
    private static final String PREFIX = "DQE";
    private static final String MIDDLE = "Q";
    private static final String SUFFIX = "E";

    /**
     * Generates an ID like DQE00284Q2000E006
     *
     * @return formatted ID string
     */
    public static String generate() {
        int firstNumber = ThreadLocalRandom.current().nextInt(0, 100000); // 5 digits
        int secondNumber = ThreadLocalRandom.current().nextInt(0, 10000); // 4 digits
        int thirdNumber = ThreadLocalRandom.current().nextInt(0, 1000);   // 3 digits

        return String.format("%s%05d%s%04d%s%03d",
                PREFIX,
                firstNumber,
                MIDDLE,
                secondNumber,
                SUFFIX,
                thirdNumber
        );
    }

    public static void main(String[] args) {
        String id = CustomIdGenerator.generate();
        System.out.println("Generated ID: " + id);
    }
}
```
